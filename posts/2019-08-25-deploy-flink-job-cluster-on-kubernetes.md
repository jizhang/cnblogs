---
title: 使用 Kubernetes 部署 Flink 应用
tags:
  - kubernetes
  - flink
categories: Big Data
date: 2019-08-25 11:02:22
---

[Kubernetes][8] 是目前非常流行的容器编排系统，在其之上可以运行 Web 服务、大数据处理等各类应用。这些应用被打包在一个个非常轻量的容器中，我们通过声明的方式来告知 Kubernetes 要如何部署和扩容这些程序，并对外提供服务。[Flink][9] 同样是非常流行的分布式处理框架，它也可以运行在 Kubernetes 之上。将两者相结合，我们就可以得到一个健壮和高可扩的数据处理应用，并且能够更安全地和其它服务共享一个 Kubernetes 集群。

![Flink on Kubernetes](/images/flink-on-kubernetes.png)

在 Kubernetes 上部署 Flink 有两种方式：会话集群（Session Cluster）和脚本集群（Job Cluster）。会话集群和独立部署一个 Flink 集群类似，只是底层资源换成了 K8s 容器，而非直接运行在操作系统上。该集群可以提交多个脚本，因此适合运行那些短时脚本和即席查询。脚本集群则是为单个脚本部署一整套服务，包括 JobManager 和 TaskManager，运行结束后这些资源也随即释放。我们需要为每个脚本构建专门的容器镜像，分配独立的资源，因而这种方式可以更好地和其他脚本隔离开，同时便于扩容或缩容。文本将以脚本集群为例，演示如何在 K8s 上运行 Flink 实时处理程序，主要步骤如下：

* 编译并打包 Flink 脚本 Jar 文件；
* 构建 Docker 容器镜像，添加 Flink 运行时库和上述 Jar 包；
* 使用 Kubernetes Job 部署 Flink JobManager 组件；
* 使用 Kubernetes Service 将 JobManager 服务端口开放到集群中；
* 使用 Kubernetes Deployment 部署 Flink TaskManager；
* 配置 Flink JobManager 高可用，需使用 ZooKeeper 和 HDFS；
* 借助 Flink SavePoint 机制来停止和恢复脚本。

<!-- more -->

## Kubernetes 实验环境

如果手边没有 K8s 实验环境，我们可以用 [Minikube][1] 快速搭建一个，以 MacOS 系统为例：

* 安装 [VirtualBox][2]，Minikube 将在虚拟机中启动 K8s 集群；
* 下载 [Minikube 程序][3]，权限修改为可运行，并加入到 PATH 环境变量中；
* 执行 `minikube start`，该命令会下载虚拟机镜像，安装 `kubelet` 和 `kubeadm` 程序，并构建一个完整的 K8s 集群。如果你在访问网络时遇到问题，可以配置一个代理，并告知 [Minikube 使用它][5]；
* 下载并安装 [kubectl 程序][4]，Minikube 已经将该命令指向虚拟机中的 K8s 集群了，所以可以直接运行 `kubectl get pods -A` 来显示当前正在运行的 K8s Pods：

```text
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   kube-apiserver-minikube            1/1     Running   0          16m
kube-system   etcd-minikube                      1/1     Running   0          15m
kube-system   coredns-5c98db65d4-d4t2h           1/1     Running   0          17m
```

## Flink 实时处理脚本示例

我们可以编写一个简单的实时处理脚本，该脚本会从某个端口中读取文本，分割为单词，并且每 5 秒钟打印一次每个单词出现的次数。以下代码是从 [Flink 官方文档][6] 上获取来的，完整的示例项目可以到 [GitHub][7] 上查看。

```java
DataStream<Tuple2<String, Integer>> dataStream = env
    .socketTextStream("192.168.99.1", 9999)
    .flatMap(new Splitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1);

dataStream.print();
```

K8s 容器中的程序可以通过 IP `192.168.99.1` 来访问 Minikube 宿主机上的服务。因此在运行上述代码之前，需要先在宿主机上执行 `nc -lk 9999` 命令打开一个端口。

接下来执行 `mvn clean package` 命令，打包好的 Jar 文件路径为 `target/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar`。

## 构建 Docker 容器镜像

Flink 提供了一个官方的容器镜像，可以从 [DockerHub][11] 上下载。我们将以这个镜像为基础，构建独立的脚本镜像，将打包好的 Jar 文件放置进去。此外，新版 Flink 已将 Hadoop 依赖从官方发行版中剥离，因此我们在打镜像时也需要包含进去。

简单看一下官方镜像的 [Dockerfile][12]，它做了以下几件事情：

* 将 OpenJDK 1.8 作为基础镜像；
* 下载并安装 Flink 至 `/opt/flink` 目录中；
* 添加 `flink` 用户和组；
* 指定入口文件，不过我们会在 K8s 配置中覆盖此项。

```Dockerfile
FROM openjdk:8-jre
ENV FLINK_HOME=/opt/flink
WORKDIR $FLINK_HOME
RUN useradd flink && \
  wget -O flink.tgz "$FLINK_TGZ_URL" && \
  tar -xf flink.tgz
ENTRYPOINT ["/docker-entrypoint.sh"]
```

在此基础上，我们编写新的 Dockerfile：

```Dockerfile
FROM flink:1.8.1-scala_2.12
ARG hadoop_jar
ARG job_jar
COPY --chown=flink:flink $hadoop_jar $job_jar $FLINK_HOME/lib/
USER flink
```

在构建镜像之前，我们需要安装 Docker 命令行工具，并将其指向 Minikube 中的 Docker 服务，这样打出来的镜像才能被 K8s 使用：

```bash
$ brew install docker
$ eval $(minikube docker-env)
```

下载 [Hadoop Jar 包][13]，执行以下命令：

```bash
$ cd /path/to/Dockerfile
$ cp /path/to/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar hadoop.jar
$ cp /path/to/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar job.jar
$ docker build --build-arg hadoop_jar=hadoop.jar --build-arg job_jar=job.jar --tag flink-on-kubernetes:0.0.1 .
```

脚本镜像打包完毕，可用于部署：

```bash
$ docker image ls
REPOSITORY           TAG    IMAGE ID      CREATED         SIZE
flink-on-kubernetes  0.0.1  505d2f11cc57  10 seconds ago  618MB
```

## 部署 JobManager

首先，我们通过创建 Kubernetes Job 对象来部署 Flink JobManager。Job 和 Deployment 是 K8s 中两种不同的管理方式，他们都可以通过启动和维护多个 Pod 来执行任务。不同的是，Job 会在 Pod 执行完成后自动退出，而 Deployment 则会不断重启 Pod，直到手工删除。Pod 成功与否是通过命令行返回状态判断的，如果异常退出，Job 也会负责重启它。因此，Job 更适合用来部署 Flink 应用，当我们手工关闭一个 Flink 脚本时，K8s 就不会错误地重新启动它。

以下是 [`jobmanager.yml`][18] 配置文件：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ${JOB}-jobmanager
spec:
  template:
    metadata:
      labels:
        app: flink
        instance: ${JOB}-jobmanager
    spec:
      restartPolicy: OnFailure
      containers:
      - name: jobmanager
        image: flink-on-kubernetes:0.0.1
        command: ["/opt/flink/bin/standalone-job.sh"]
        args: ["start-foreground",
               "-Djobmanager.rpc.address=${JOB}-jobmanager",
               "-Dparallelism.default=1",
               "-Dblob.server.port=6124",
               "-Dqueryable-state.server.ports=6125"]
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob
        - containerPort: 6125
          name: query
        - containerPort: 8081
          name: ui
```

* `${JOB}` 变量可以使用 `envsubst` 命令来替换，这样同一份配置文件就能够为多个脚本使用了；
* 容器的入口修改为了 `standalone-job.sh`，这是 Flink 的官方脚本，会以前台模式启动 JobManager，扫描类加载路径中的 `Main-Class` 作为脚本入口，我们也可以使用 `-j` 参数来指定完整的类名。之后，这个脚本会被自动提交到集群中。
* JobManager 的 RPC 地址修改为了 [Kubernetes Service][14] 的名称，我们将在下文创建。集群中的其他组件将通过这个名称来访问 JobManager。
* Flink Blob Server & Queryable State Server 的端口号默认是随机的，为了方便将其开放到集群中，我们修改为了固定端口。

使用 `kubectl` 命令创建对象，并查看状态：

```bash
$ export JOB=flink-on-kubernetes
$ envsubst <jobmanager.yml | kubectl create -f -
$ kubectl get pod
NAME                                   READY   STATUS    RESTARTS   AGE
flink-on-kubernetes-jobmanager-kc4kq   1/1     Running   0          2m26s
```

随后，我们创建一个 K8s Service 来将 JobManager 的端口开放出来，以便 TaskManager 前来注册：

`service.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${JOB}-jobmanager
spec:
  selector:
    app: flink
    instance: ${JOB}-jobmanager
  type: NodePort
  ports:
  - name: rpc
    port: 6123
  - name: blob
    port: 6124
  - name: query
    port: 6125
  - name: ui
    port: 8081
```

这里 `type: NodePort` 是必要的，因为通过这项配置，我们可以在 K8s 集群之外访问 JobManager UI 和 RESTful API。

```bash
$ envsubst <service.yml | kubectl create -f -
$ kubectl get service
NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                      AGE
flink-on-kubernetes-jobmanager   NodePort    10.109.78.143   <none>        6123:31476/TCP,6124:32268/TCP,6125:31602/TCP,8081:31254/TCP  15m
```

我们可以看到，Flink Dashboard 开放在了虚拟机的 31254 端口上。Minikube 提供了一个命令，可以获取到 K8s 服务的访问地址：

```bash
$ minikube service $JOB-jobmanager --url
http://192.168.99.108:31476
http://192.168.99.108:32268
http://192.168.99.108:31602
http://192.168.99.108:31254
```

## 部署 TaskManager

`taskmanager.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${JOB}-taskmanager
spec:
  selector:
    matchLabels:
      app: flink
      instance: ${JOB}-taskmanager
  replicas: 1
  template:
    metadata:
      labels:
        app: flink
        instance: ${JOB}-taskmanager
    spec:
      containers:
      - name: taskmanager
        image: flink-on-kubernetes:0.0.1
        command: ["/opt/flink/bin/taskmanager.sh"]
        args: ["start-foreground", "-Djobmanager.rpc.address=${JOB}-jobmanager"]
```

通过修改 `replicas` 配置，我们可以开启多个 TaskManager。镜像中的 `taskmanager.numberOfTaskSlots` 参数默认为 `1`，这也是我们推荐的配置，因为扩容缩容方面的工作应该交由 K8s 来完成，而非直接使用 TaskManager 的槽位机制。

至此，Flink 脚本集群已经在运行中了。我们在之前已经打开的 `nc` 命令窗口中输入一些文本：

```bash
$ nc -lk 9999
hello world
hello flink
```

打开另一个终端，查看 TaskManager 的标准输出日志：

```bash
$ kubectl logs -f -l instance=$JOB-taskmanager
(hello,2)
(flink,1)
(world,1)
```

## 开启高可用模式

可用性方面，上述配置中的 TaskManager 如果发生故障退出，K8s 会自动进行重启，Flink 会从上一个 Checkpoint 中恢复工作。但是，JobManager 仍然存在单点问题，因此需要开启 [HA 模式][15]，配合 ZooKeeper 和分布式文件系统（如 HDFS）来实现 JobManager 的高可用。在独立集群中，我们需要运行多个 JobManager，作为主备服务器。然而在 K8s 模式下，我们只需开启一个 JobManager，当其异常退出后，K8s 会负责重启，新的 JobManager 将从 ZooKeeper 和 HDFS 中读取最近的工作状态，自动恢复运行。

开启 HA 模式需要修改 JobManager 和 TaskManager 的启动命令：

`jobmanager-ha.yml`

```yaml
command: ["/opt/flink/bin/standalone-job.sh"]
args: ["start-foreground",
       "-Djobmanager.rpc.address=${JOB}-jobmanager",
       "-Dparallelism.default=1",
       "-Dblob.server.port=6124",
       "-Dqueryable-state.server.ports=6125",
       "-Dhigh-availability=zookeeper",
       "-Dhigh-availability.zookeeper.quorum=192.168.99.1:2181",
       "-Dhigh-availability.zookeeper.path.root=/flink",
       "-Dhigh-availability.cluster-id=/${JOB}",
       "-Dhigh-availability.storageDir=hdfs://192.168.99.1:9000/flink/recovery",
       "-Dhigh-availability.jobmanager.port=6123",
       ]
```

`taskmanager-ha.yml`

```yaml
command: ["/opt/flink/bin/taskmanager.sh"]
args: ["start-foreground",
       "-Dhigh-availability=zookeeper",
       "-Dhigh-availability.zookeeper.quorum=192.168.99.1:2181",
       "-Dhigh-availability.zookeeper.path.root=/flink",
       "-Dhigh-availability.cluster-id=/${JOB}",
       "-Dhigh-availability.storageDir=hdfs://192.168.99.1:9000/flink/recovery",
       ]
```

* 准备好 ZooKeeper 和 HDFS 测试环境，该配置中使用的是宿主机上的 `2181` 和 `9000` 端口；
* Flink 集群基本信息会存储在 ZooKeeper 的 `/flink/${JOB}` 目录下；
* Checkpoint 数据会存储在 HDFS 的 `/flink/recovery` 目录下。使用前，请先确保 Flink 有权限访问 HDFS 的 `/flink` 目录；
* `jobmanager.rpc.address` 选项从 TaskManager 的启动命令中去除了，是因为在 HA 模式下，TaskManager 会通过访问 ZooKeeper 来获取到当前 JobManager 的连接信息。需要注意的是，HA 模式下的 JobManager RPC 端口默认是随机的，我们需要使用 `high-availability.jobmanager.port` 配置项将其固定下来，方便在 K8s Service 中开放。

## 管理 Flink 脚本

我们可以通过 RESTful API 来与 Flink 集群交互，其端口号默认与 Dashboard UI 一致。在宿主机上安装 Flink 命令行工具，传入 `-m ` 参数来指定目标集群：

```bash
$ bin/flink list -m 192.168.99.108:30206
------------------ Running/Restarting Jobs -------------------
24.08.2019 12:50:28 : 00000000000000000000000000000000 : Window WordCount (RUNNING)
--------------------------------------------------------------
```

在 HA 模式下，Flink 脚本 ID 默认为 `00000000000000000000000000000000`，我们可以使用这个 ID 来手工停止脚本，并生成一个 [SavePoint][16] 快照：

```bash
$ bin/flink cancel -m 192.168.99.108:30206 -s hdfs://192.168.99.1:9000/flink/savepoints/ 00000000000000000000000000000000
Cancelled job 00000000000000000000000000000000. Savepoint stored in hdfs://192.168.99.1:9000/flink/savepoints/savepoint-000000-f776c8e50a0c.
```

执行完毕后，可以看到 K8s Job 对象的状态变为了已完成：

```bash
$ kubectl get job
NAME                             COMPLETIONS   DURATION   AGE
flink-on-kubernetes-jobmanager   1/1           4m40s      7m14s
```

重新启动脚本前，我们需要先将配置从 K8s 中删除：

```bash
$ kubectl delete job $JOB-jobmanager
$ kubectl delete deployment $JOB-taskmanager
```

然后在 JobManager 的启动命令中加入 `--fromSavepoint` 参数：

```yaml
command: ["/opt/flink/bin/standalone-job.sh"]
args: ["start-foreground",
       ...
       "--fromSavepoint", "${SAVEPOINT}",
       ]
```

使用刚才得到的 SavePoint 路径替换该变量，并启动 JobManager：

```bash
$ export SAVEPOINT=hdfs://192.168.99.1:9000/flink/savepoints/savepoint-000000-f776c8e50a0c
$ envsubst <jobmanager-savepoint.yml | kubectl create -f -
```

需要注意的是，SavePoint 必须和 HA 模式配合使用，因为当 JobManager 异常退出、K8s 重启它时，都会传入 `--fromSavepoint`，使脚本进入一个异常的状态。而在开启 HA 模式时，JobManager 会优先读取最近的 CheckPoint 并从中恢复，忽略命令行中传入的 SavePoint。

### 扩容

有两种方式可以对 Flink 脚本进行扩容。第一种方式是用上文提到的 SavePoint 机制：手动关闭脚本，并使用新的 `replicas` 和 `parallelism.default` 参数进行重启；另一种方式则是使用 `flink modify` 命令行工具，该工具的工作机理和人工操作类似，也是先用 SavePoint 停止脚本，然后以新的并发度启动。在使用第二种方式前，我们需要在启动命令中指定默认的 SavePoint 路径：

```yaml
command: ["/opt/flink/bin/standalone-job.sh"]
args: ["start-foreground",
       ...
       "-Dstate.savepoints.dir=hdfs://192.168.99.1:9000/flink/savepoints/",
       ]
```

然后，使用 `kubectl scale` 命令调整 TaskManager 的个数；

```bash
$ kubectl scale --replicas=2 deployment/$JOB-taskmanager
deployment.extensions/flink-on-kubernetes-taskmanager scaled
```

最后，使用 `flink modify` 调整脚本并发度：

```bash
$ bin/flink modify 755877434b676ce9dae5cfb533ed7f33 -m 192.168.99.108:30206 -p 2
Modify job 755877434b676ce9dae5cfb533ed7f33.
Rescaled job 755877434b676ce9dae5cfb533ed7f33. Its new parallelism is 2.
```

但是，因为存在一个尚未解决的 [Issue][17]，我们无法使用 `flink modify` 命令来对 HA 模式下的 Flink 集群进行扩容，因此还请使用人工的方式操作。

## Flink 将原生支持 Kubernetes

Flink 有着非常活跃的开源社区，他们不断改进自身设计（[FLIP-6][10]），以适应现今的云原生环境。他们也注意到了 Kubernetes 的蓬勃发展，对 K8s 集群的原生支持也在开发中。我们知道，Flink 可以直接运行在 YARN 或 Mesos 资源管理框架上。以 YARN 为例，Flink 首先启动一个 ApplicationMaster，作为 JobManager，分析提交的脚本需要多少资源，并主动向 YARN ResourceManager 申请，开启对应的 TaskManager。当脚本的并行度改变后，Flink 会自动新增或释放 TaskManager 容器，达到扩容缩容的目的。这种主动管理资源的模式，社区正在开发针对 Kubernetes 的版本（[FLINK-9953][19]），今后我们便可以使用简单的命令来将 Flink 部署到 K8s 上了。

此外，另一种资源管理模式也在开发中，社区称为响应式容器管理（[FLINK-10407 Reactive container mode][20]）。简单来说，当 JobManager 发现手中有多余的 TaskManager 时，会自动将运行中的脚本扩容到相应的并发度。以上文中的操作为例，我们只需使用 `kubectl scale` 命令修改 TaskManager Deployment 的 `replicas` 个数，就能够达到扩容和缩容的目的，无需再执行 `flink modify`。相信不久的将来我们就可以享受到这些便利的功能。

## 参考资料

* https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/deployment/kubernetes.html
* https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/
* https://jobs.zalando.com/tech/blog/running-apache-flink-on-kubernetes/
* https://www.slideshare.net/tillrohrmann/redesigning-apache-flinks-distributed-architecture-flink-forward-2017
* https://www.slideshare.net/tillrohrmann/future-of-apache-flink-deployments-containers-kubernetes-and-more-flink-forward-2019-sf

[1]: https://github.com/kubernetes/minikube
[2]: https://www.virtualbox.org
[3]: https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
[4]: https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/darwin/amd64/kubectl
[5]: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
[6]: https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#example-program
[7]: https://github.com/jizhang/blog-demo/tree/master/flink-on-kubernetes
[8]: https://kubernetes.io/
[9]: https://flink.apache.org/
[10]: https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077
[11]: https://hub.docker.com/_/flink
[12]: https://github.com/docker-flink/docker-flink/blob/master/1.8/scala_2.12-debian/Dockerfile
[13]: https://repo.maven.apache.org/maven2/org/apache/flink/flink-shaded-hadoop-2-uber/2.8.3-7.0/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar
[14]: https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
[15]: https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/jobmanager_high_availability.html
[16]: https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/state/savepoints.html
[17]: https://issues.apache.org/jira/browse/FLINK-11997
[18]: https://github.com/jizhang/blog-demo/blob/master/flink-on-kubernetes/docker/jobmanager.yml
[19]: https://issues.apache.org/jira/browse/FLINK-9953
[20]: https://issues.apache.org/jira/browse/FLINK-10407
