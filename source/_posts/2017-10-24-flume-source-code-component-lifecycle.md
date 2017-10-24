---
title: Flume 源码解析：组件生命周期
tags:
  - flume
  - java
  - source code
categories: Big Data
date: 2017-10-24 09:18:26
---


[Apache Flume](https://flume.apache.org/) 是数据仓库体系中用于做实时 ETL 的工具。它提供了丰富的数据源和写入组件，这些组件在运行时都由 Flume 的生命周期管理机制进行监控和维护。本文将对这部分功能的源码进行解析。

## 项目结构

Flume 的源码可以从 GitHub 上下载。它是一个 Maven 项目，我们将其导入到 IDE 中以便更好地进行源码阅读。以下是代码仓库的基本结构：

```
/flume-ng-node
/flume-ng-code
/flume-ng-sdk
/flume-ng-sources/flume-kafka-source
/flume-ng-channels/flume-kafka-channel
/flume-ng-sinks/flume-hdfs-sink
```

## 程序入口

Flume Agent 的入口 `main` 函数位于 `flume-ng-node` 模块的 `org.apache.flume.node.Application` 类中。下列代码是该函数的摘要：

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

启动过程说明如下：

1. 使用 `commons-cli` 对命令行参数进行解析，提取 Agent 名称、配置信息读取方式及其路径信息；
2. 配置信息可以通过文件或 ZooKeeper 的方式进行读取，两种方式都支持热加载，即我们不需要重启 Agent 就可以更新配置内容：
    * 基于文件的配置热加载是通过一个后台线程对文件进行轮询实现的；
    * 基于 ZooKeeper 的热加载则是使用了 Curator 的 `NodeCache` 模式，底层是 ZooKeeper 原生的监听（Watch）特性。
3. 如果配置热更新是开启的（默认开启），配置提供方 `ConfigurationProvider` 就会将自身注册到 Agent 程序的组件列表中，并在 `Application#start` 方法调用后，由 `LifecycleSupervisor` 类进行启动和管理，加载和解析配置文件，从中读取组件列表。
4. 如果热更新未开启，则配置提供方将在启动时立刻读取配置文件，并由 `LifecycleSupervisor` 启动和管理所有组件。
5. 最后，`main` 会调用 `Runtime#addShutdownHook`，当 JVM 关闭时（SIGTERM 或者 Ctrl+C），`Application#stop` 会被用于关闭 Flume Agent，使各组件优雅退出。

<!-- more -->

## 配置重载

在 `PollingPropertiesFileConfigurationProvider` 类中，当文件内容更新时，它会调用父类的 `AbstractConfigurationProvider#getConfiguration` 方法，将配置内容解析成 `MaterializedConfiguration` 实例，这个对象实例中包含了数据源（Source）、目的地（Sink）、以及管道（Channel）组件的所有信息。随后，这个轮询线程会通过 Guava 的 `EventBus` 机制通知 `Application` 类配置发生了更新，从而触发 `Application#handleConfigurationEvent` 方法，重新加载所有的组件。

```java
// Application 类
@Subscribe
public synchronized void handleConfigurationEvent(MaterializedConfiguration conf) {
  stopAllComponents();
  startAllComponents(conf);
}

// PollingPropertiesFileConfigurationProvider$FileWatcherRunnable 内部类
@Override
public void run() {
  eventBus.post(getConfiguration());
}
```

## 启动组件

组件启动的流程位于 `Application#startAllComponents` 方法中。这个方法接收到新的组件信息后，首先将启动所有的 `Channel`，然后启动 `Sink` 和 `Source`。

```java
private void startAllComponents(MaterializedConfiguration materializedConfiguration) {
  this.materializedConfiguration = materializedConfiguration;
  for (Entry<String, Channel> entry :
      materializedConfiguration.getChannels().entrySet()) {
    supervisor.supervise(entry.getValue(),
        new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
  }
  //  等待所有管道启动完毕
  for (Channel ch : materializedConfiguration.getChannels().values()) {
    while (ch.getLifecycleState() != LifecycleState.START
        && !supervisor.isComponentInErrorState(ch)) {
      Thread.sleep(500);
    }
  }
  // 相继启动目的地和数据源组件
}
```

`LifecycleSupervisor` 类（代码中的 `supervisor` 变量）可用于管理实现了 `LifecycleAware` 接口的组件。该类会初始化一个 `MonitorRunnable`，每三秒轮询一次组件状态，通过 `LifecycleAware#start` 和 `stop` 方法，保证其始终处于 `desiredState` 变量所指定的状态。

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

当 JVM 关闭时，钩子函数会调用 `Application#stop` 方法，进而调用 `LifecycleSupervisor#stop`。该方法首先停止所有的 `MonitorRunnable` 线程，将组件目标状态置为 `STOP`，并调用 `LifecycleAware#stop` 方法命其优雅终止。

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

对于单个组件的生命周期，我们以 `KafkaSource` 为例：

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

`KafkaSource` 被定义成轮询式的数据源，也就是说我们需要使用一个线程不断对其进行轮询，查看是否有数据可以供处理：

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

  // 轮询线程
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

`AbstractPollableSource` 和 `SourceRunner` 都实现了 `LifecycleAware` 接口，因此都有 `start` 和 `stop` 方法。但是，只有 `SourceRunner` 会由 `LifecycleSupervisor` 管理，`PollableSource` 则是附属于 `SourceRunner` 的一个组件。我们可以在 `AbstractConfigurationProvider#loadSources` 中看到配置关系：

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
