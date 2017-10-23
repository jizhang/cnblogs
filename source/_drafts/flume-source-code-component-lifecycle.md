---
title: Flume 源码解析：组件生命周期
tags:
  - flume
  - java
  - source code
categories: Big Data
---

[Apache Flume](https://flume.apache.org/) is a real-time ETL tool for data warehouse platform. It consists of different types of components, and during runtime all of them are managed by Flume's lifecycle and supervisor mechanism. This article will walk you through the source code of Flume's component lifecycle management.

## 项目结构

Flume's source code can be downloaded from GitHub. It's a Maven project, so we can import it into an IDE for efficient code reading. The following is the main structure of the project:

```
/flume-ng-node
/flume-ng-code
/flume-ng-sdk
/flume-ng-sources/flume-kafka-source
/flume-ng-channels/flume-kafka-channel
/flume-ng-sinks/flume-hdfs-sink
```

## 程序入口

The `main` entrance of Flume agent is in the `org.apache.flume.node.Application` class of `flume-ng-node` module. Following is an abridged source code:

```java
public class Application {
  public static void main(String[] args) {
    CommandLineParser parser = new GnuParser();
    if (isZkConfigured) {
      if (reload) {
        PollingZooKeeperConfigurationProvider zookeeperConfigurationProvider;
        components.add(zookeeperConfigurationProvider);
      } else {
        StaticZooKeeperConfigurationProvider zookeeperConfigurationProvider;
        application.handleConfigurationEvent();
      }
    } else {
      // PropertiesFileConfigurationProvider
    }
    application.start();
    Runtime.getRuntime().addShutdownHook(new Thread("agent-shutdown-hook") {
      @Override
      public void run() {
        appReference.stop();
      }
    });
  }
}
```

The process can be illustrated as follows:

1. Parse command line arguments with `commons-cli`, including the Flume agent's name, configuration method and path.
2. Configurations can be provided via properties file or ZooKeeper. Both provider support live-reload, i.e. we can update component settings without restarting the agent.
    * File-based live-reload is implemented by using a background thread that checks the last modification time of the file.
    * ZooKeeper-based live-reload is provided by Curator's `NodeCache` recipe, which uses ZooKeeper's *watch* functionality underneath.
3. If live-reload is on (by default), configuration providers will add themselves into the application's component list, and after calling `Application#start`, a `LifecycleSupervisor` will start the provider, and trigger the reload event to parse the configuration and load all defined components.
4. If live-reload is off, configuration providers will parse the file immediately and start all components, also supervised by `LifecycleSupervisor`.
5. Finally add a JVM shutdown hook by `Runtime#addShutdownHook`, which in turn invokes `Application#stop` to shutdown the Flume agent.

<!-- more -->

## 配置重载

In `PollingPropertiesFileConfigurationProvider`, when it detects file changes, it will invoke the `AbstractConfigurationProvider#getConfiguration` method to parse the configuration file into an `MaterializedConfiguration` instance, which contains the source, sink, and channel definitions. And then, the polling thread send an event to `Application` via a Guava's `EventBus` instance, which effectively invokes the `Application#handleConfigurationEvent` method to reload all components.

```java
// Application class
@Subscribe
public synchronized void handleConfigurationEvent(MaterializedConfiguration conf) {
  stopAllComponents();
  startAllComponents(conf);
}

// PollingPropertiesFileConfigurationProvider$FileWatcherRunnable
@Override
public void run() {
  eventBus.post(getConfiguration());
}
```

## 启动组件

The starting process lies in `Application#startAllComponents`. The method accepts a new set of components, starts the `Channel`s first, followed by `Sink`s and `Source`s.

```java
private void startAllComponents(MaterializedConfiguration materializedConfiguration) {
  this.materializedConfiguration = materializedConfiguration;
  for (Entry<String, Channel> entry :
      materializedConfiguration.getChannels().entrySet()) {
    supervisor.supervise(entry.getValue(),
        new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
  }
  //  Wait for all channels to start.
  for (Channel ch : materializedConfiguration.getChannels().values()) {
    while (ch.getLifecycleState() != LifecycleState.START
        && !supervisor.isComponentInErrorState(ch)) {
      Thread.sleep(500);
    }
  }
  // Start and supervise sinkds and sources
}
```

The `LifecycleSupervisor` manages instances that implement `LifecycleAware` interface. Supervisor will schedule a `MonitorRunnable` instance with a fixed delay (3 secs), which tries to convert a `LifecycleAware` instance into its `desiredState`, by calling `LifecycleAware#start` or `stop`.

```java
public static class MonitorRunnable implements Runnable {
  @Override
  public void run() {
    if (!lifecycleAware.getLifecycleState().equals(
        supervisoree.status.desiredState)) {
      switch (supervisoree.status.desiredState) {
        case START:
          lifecycleAware.start();
          break;
        case STOP:
          lifecycleAware.stop();
      }
    }
  }
}
```

## 停止组件

When JVM is shutting down, the hook invokes `Application#stop`, which calls `LifecycleSupervisor#stop`, that first shutdowns the `MonitorRunnable`s' executor pool, and changes all components' desired status to `STOP`, waiting for them to fully shutdown.

```java
public class LifecycleSupervisor implements LifecycleAware {
  @Override
  public synchronized void stop() {
    monitorService.shutdown();
    for (final Entry<LifecycleAware, Supervisoree> entry :
        supervisedProcesses.entrySet()) {
      if (entry.getKey().getLifecycleState().equals(LifecycleState.START)) {
        entry.getValue().status.desiredState = LifecycleState.STOP;
        entry.getKey().stop();
      }
    }
  }
}
```

## Source 与 SourceRunner

Take `KafkaSource` for an instance, we shall see how agent supervises source components, and the same thing happens to sinks and channels.

```java
public class KafkaSource extends AbstractPollableSource {
  @Override
  protected void doStart() throws FlumeException {
    consumer = new KafkaConsumer<String, byte[]>(kafkaProps);
    it = consumer.poll(1000).iterator();
  }

  @Override
  protected void doStop() throws FlumeException {
    consumer.close();
  }
}
```

`KafkaSource` is a pollable source, which means it needs a runner thread to constantly poll for more data to process.

```java
public class PollableSourceRunner extends SourceRunner {
  @Override
  public void start() {
    source.start();
    runner = new PollingRunner();
    runnerThread = new Thread(runner);
    runnerThread.start();
    lifecycleState = LifecycleState.START;
  }

  @Override
  public void stop() {
    runnerThread.interrupt();
    runnerThread.join();
    source.stop();
    lifecycleState = LifecycleState.STOP;
  }

  public static class PollingRunner implements Runnable {
    @Override
    public void run() {
      while (!shouldStop.get()) {
        source.process();
      }
    }
  }
}
```

Both `AbstractPollableSource` and `SourceRunner` are subclass of `LifecycleAware`, which means they have `start` and `stop` methods for supervisor to call. In this case, `SourceRunner` is the component that Flume agent actually supervises, and `PollableSource` is instantiated and managed by `SourceRunner`. Details lie in `AbstractConfigurationProvider#loadSources`:

```java
private void loadSources(Map<String, SourceRunner> sourceRunnerMap) {
  Source source = sourceFactory.create();
  Configurables.configure(source, config);
  sourceRunnerMap.put(comp.getComponentName(),
      SourceRunner.forSource(source));
}
```

## 参考资料

* https://github.com/apache/flume
* https://flume.apache.org/FlumeUserGuide.html
* https://kafka.apache.org/0100/javadoc/index.html
