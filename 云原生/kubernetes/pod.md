# 彻底搞懂Pod
## 什么是Pod
### Pod的抽象定义
在各种文章和书籍中,  都可以看到一句话, Pod是Kubernetes项目的原子或者最小调度单位.  还有另外一种定义,   Pod是Kubernetes项目中最小的API对象.  这种定义在了解Kubernetes之后,  总结出来的一句话,  是一个抽象出来的概念.
### 面临的问题
在讨论这个问题之前,  需要明白一个基本的原理, 容器的本质到底是什么? 容器的本质是进程.  容器就是未来云计算系统中的进程,  容器镜像就是这个系统里的“.exe”安装包.  
那么Kubernetes是什么?  Kubernetes是操作系统.  kubernetes是负责编排、管理容器的工具.
在一个操作系统里,  进程并不是独自地运行,  而是以进程组的方式,  合理的组织在一起.  这些密切合作的进程会组成进程组,  把它们部署在同一台机器上.  如果不在同一台机器上,  它们之间基于Socket的通信和文件交换就会出问题(rsyslogd\imklog\imuxsock).
如果把rsyslogd容器化部署,  并且它们都会占用一定的资源,  有如下部署方式:
1. 先部署其中的一个容器,  然后使用亲密性约束,  让其他相关的容器,  也会部署在同一个机器上,  这样做的问题是,  如果在部署完某些容器之后,  机器的资源不够了,  这时就无法完成部署.
2. 资源囤积的机制,  会在所有设置了Affinity约束的任务都达到时,  才开始对它们统一进程调度.  资源囤积带来了不可避免的调度效率损失和死锁的可能性.
3. 乐观调度.  先不管这些冲突,  通过精心设计的回滚机制在出现了冲突之后解决问题.  乐观调度的复杂度很高.
在Kubernetes中,  如果把rsyslogd容器化部署,  会把这几个容器放在一个Pod中,  部署这个进程组所需要的资源会按照Pod的需求来计算,  这样问题就引刃而解了.
像这样容器间的紧密协作，我们可以称为“超亲密关系”。这些具有“超亲密关系”容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。
另外Pod的设计中使用到了容器设计模式,  会在下面提到.
### Pod的实现原理
Pod只是一个逻辑概念,  Kubernetes真正处理的,  还是宿主机操作系统上Linux容器的Namespace和Cgroups,  并不存在一个所谓的Pod的边界或者隔离环境.
Pod的创建只是创建了一组共享某些资源的容器.  在创建Pod时,  会使用一个叫做Infra容器,  这个容器永远都是第一个被创建的容器,  而其他用户定义的容器,  则通过Join Network Namespace的方式,  与Infra容器关联在一起. 所以Pod里的所有容器,  共享的是同一个Network Namesapce,  并且可以声明共享同一个Volume. 就像下面这个图:
![infra-and-containers](./assets/infra-and-container.png)
### 什么的Infra容器
Infra容器是一个使用汇编编写的、永远处于“暂停”状态的容器,  这个容器占用极少的机缘,  解压后大小也只有100~200KB左右,  是一个非常特殊的镜像. Infra容器负责是Network Namesapce的管理,  对于同一个Pod里面的所有用户容器来说,  它们的进出流量都是通过Infra容器完成的,  所以在为Kubernetes开发网络插件时,  考虑得应该是如何配置这个Pod的Network Namespace.
其实在docker中,  可以通过命令来实现两个容器共享网络的目的:
 ```
$ docker run --net=B --volumes-from=B --name=A image-A ...
 ```
 如果这样做的话,  容器B就必须比容器A先启动,  这样一个Pod里的多个容器就不是对等关系,  而是拓扑关系了.
 这个特殊的镜像叫做: k8s.gcr.io/pause

### Pod中Volume的实现
Kubernetes项目把所有的Volume定义都设计在Pod层级,  跟网络设计类似, 这样,  一个Volume对应的宿主机目录对于Pod来说就只有一个,  Pod里的容器只要声明挂载这个Volume,  就一定可以共享这个Volume对应的宿主机目录.
### 容器设计模式
容器设计模式就是当用户想在一个容器里跑多个功能并不相关的应用时,  应该优先考虑它们是不是更应该被描述成一个Pod里的多个容器.  容器设计模式也有很多种.
## 如何使用Pod
### 如何创建一个Pod
在创建Pod时,  一般使用声明式的方式,  也就是使用文件来描述要部署的应用是什么样子. 例如 simple-pod.yaml:

```
apiVersion: v1 //调用v1版本的api创建Pod
kind: Pod  //声明了一个类型为Pod的资源
metadata:  //关于这个Pod的metadata	
  name: nginx
spec:
  containers;
  - name: nginx
     image: nginx:1.14.2
     ports:
     - containerPort: 80 
```
在这里声明了一个Pod类型的资源,  使用如下命令创建Pod

```
kubectl apply -f 
```
####  Kuberneters中如何使用Pod
在kubernetes中,  一般不直接去创建Pod.  我们先来考虑如下问题:

1. 如果直接去创建单个Pod,  当请求量太大时,  一个Pod支持不了,  这时应该去扩容.
2. 如果Pod中的程序crash了,  需要去重新启动Pod
3. 在应用程序有新的改动时,  需要部署新的版本,  这时使用Pod该怎么部署

在考虑到上面的问题时,  会发现如果使用单个Pod来部署的话,  是无法满足需求的,  所以在Kubernetes中,  会有其他类型的资源来管理Pod, 例如:

1. 如果想要有多个实例来提供服务时,  可以使用ReplicationController和ReplicationSet
2. 如果想要有滚动升级时,  可以使用Deployment
3. 如果应用程序需要有状态,  就使用StatefulSet
4. 如果应用程序需要定时工作,  可以使用Cronjob.

我们在使用Pod时, 主要有以下两种方式:

1. Pod中有一个Container. 在这种情况下，您可以将 Pod 视为单个容器的包装器； Kubernetes 管理 Pod，而不是直接管理容器。
2. Pod中有多个容器, 并且它们紧密合作.  在这种情况下,  多个容器化多个应用程序,  并且把他们放在一个Pod中,  这些应用程序紧密合作,  例如它们需要共享资源, 主应用程序使用文件, sidecar容器更行文件. Pod将这些容器、存储资源和临时网络身份包装再一起作为一个单元.

在Kubernetes中, 通过都需要横向扩展应用程序,  就需要使用多个Pod,  这被称为Replication(副本).  通常使用其他控制器资源来管理Replication.
#### Pod拉镜像的策略
spec中有个字段是imagePullPolicy,  这个地段来控制如何拉去镜像.
* IfNotPresent:  默认值,  镜像在宿主机上不存在时才拉取.
* Always:  每次创建Pod都会重新拉取一次镜像.
* Never: Pod永远不会主动拉去这个镜像.
#### Pod的分类
Pod有两种类型,  普通Pod和静态Pod.

普通 Pod 一旦被创建，就会被放入到 etcd 中存储，随后会被 Kubernetes Master 调度到某 个具体的 Node 上并进行绑定，随后该 Pod 对应的 Node 上的 kubelet 进程实例化成一组相 关的 Docker 容器并启动起来。在默认情 况下，当 Pod 里某个容器停止时，Kubernetes 会 自动检测到这个问题并且重新启动这个 Pod 里某所有容器， 如果 Pod 所在的 Node 宕机， 则会将这个 Node 上的所有 Pod 重新调度到其它节点上。

静态 Pod 是由 kubelet 进行管理的仅存在于特定 Node 上的 Pod,它们不能通过 API Server 进行管理，无法与 ReplicationController、Deployment 或 DaemonSet 进行关联，并且 kubelet 也无法对它们进行健康检查。

#### Pod的生命周期
##### Pod的生命期
Pod在其生命周期中智慧被调度一次.  一旦Pod被调度到某个节点,  Pod会一直在该节点运行,  直到Pod停止或者被终止.
##### Pod的阶段
* Pending.  Pod已被Kubernetes系统接受,  但有一个或者多个容器尚未创建亦未运行.  此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间.
* Running. Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。
* Successd. Pod 中的所有容器都已成功终止，并且不会再重启.
* Faild. Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止.
* Unknown. 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。

#### Pod的重启机制
可以为Pod配置重启策略,  Pod的spec中包含一个restartPolicy字段,  其可能取值包括Always、OnFailure和Never.  默认值是Always. restartPolicy 适用于 Pod 中的所有容器

* Always:  当容器终止退出后,  总是重启容器,  默认策略.
* OnFailure:  当容器异常退出(退出状态码非0)时,  才重启容器.
* Never: 当容器终止退出,  从不重启容器.

#### Pod的健康检查
在Kubernetes中对Pod有一个容器探针的机制(probe).  Probe是由Kubelet对容器执行的定期诊断,  这个诊断既可以是在容器内执行代码,  也可以发出一个网络请求.
##### 探针检查方式
* exec: 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
* grpc: 使用 gRPC 执行一个远程过程调用。 目标应该实现 gRPC健康检查。 如果响应的状态是 "SERVING"，则认为诊断成功。 gRPC 探针是一个 alpha 特性，只有在你启用了 "GRPCContainerProbe" 特性门控时才能使用。
* httpGet: 对容器的 IP 地址上指定端口和路径执行 HTTP GET 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。
* tcpSocket: 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。 如果远程系统（容器）在打开连接后立即将其关闭，这算作是健康的。

##### 探测结果
每次探测都将获得以下三种结果之一：

* Success: 容器通过了诊断。
* Failure: 容器未通过诊断。
* Unknown: 诊断失败，因此不会采取任何行动。

##### 探针类型
* livennessProbe: 指示容器是否正在运行。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其重启策略决定未来。如果容器不提供存活探针， 则默认状态为 Success。
* readinessProbe: 指示容器是否准备好为请求提供服务。如果检查失败, Kubernetes会把Pod从service endpoint中剔除.  初始延迟之前的就绪态的状态值默认为 Failure。 如果容器不提供就绪态探针，则默认状态为 Success。
* startupProbe: 指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，kubelet 将杀死容器，而容器依其 重启策略进行重启。 如果容器没有提供启动探测，则默认状态为 Success。

##### 何时使用探针
Kubelet 使用 liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，liveness 探针将捕获到 deadlock，重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去.  如果容器中的进程能够在遇到问题或不健康的情况下自行崩溃,  则不一定需要存活探针; kubelet 将根据 Pod 的restartPolicy 自动执行修复操作。

Kubelet 使用 readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当 Pod 中的容器都处于就绪状态时 kubelet 才会认定该 Pod 处于就绪状态。该信号的作用是控制哪些 Pod 应该作为 service 的后端。如果 Pod 处于非就绪状态，那么它们将会被从 service 的 load balancer 中移除。

#### Pod的资源限制
Kubernetes提供了对机制限制Pod的资源使用情况,  有如下字段:

* spec.containers[].resources.limits.cpu
* spec.containers[].resources.limits.memory
* spec.containers[].resources.requests.cpu
* spec.containers[].resources.requests.memory

Kubernetes根据request中资源描述来对Pod进行调度,  会查看Node中的资源是否能够满足满足,  如果满足的话,  就可以把该Pod调度到该Node上.  如果Node的资源足够多,  超过了request的需求,  那么Pod可以申请使用更多资源.

Kubernetes不允许Pod使用超过limit中所描述的资源,  kubelet或者container runtime会保证这个约束生效.  如果Pod资源使用超出limit中的描述, Kubernetes会停止该Pod.

Pod所需要的资源是各个容器所需要的资源的总和.

#####  不为设置资源限制
如果不为Container制定内存限制,  会有以下情况:

* Container 对其使用的内存量没有上限。 容器可以使用它正在运行的节点上的所有可用内存，这样的容器也有可能被kill掉。 此外，在 OOM Kill 的情况下，没有资源限制的容器将有更大的机会被杀死。
* Container 在具有默认内存限制的命名空间中运行，并且自动为 Container 分配了默认限制。 集群管理员可以使用 LimitRange 来指定内存限制的默认值。

如果不为Container设置CPU限制,  会有以下情况:

* Container 可以使用的 CPU 资源没有上限。 容器可以使用运行它的节点上所有可用的 CPU 资源。
* Container 在具有默认 CPU 限制的命名空间中运行，并且自动为 Container 分配默认限制。 集群管理员可以使用 LimitRange 来指定 CPU 限制的默认值。

##### 如果只设置limit,  不设置request
如果您为 Container 指定 CPU limit但未指定 CPU request，Kubernetes 会自动分配与limit匹配的 CPU request。 同样，如果一个Container指定了自己的内存limit，但没有指定内存request，Kubernetes会自动分配一个匹配该limit的内存request。


### Init容器
在Pod中可以运行多个容器,  kubernetes中提供了一种init容器的机制.  这类容器是在网络和数据卷初始化之后,  在常规能够持续运行的容器之前去运行.  当所有的Init容器运行完成之后,  Kubernetes才会为Pod初始化常规应用程序容器.

如果Init失败,  kubelet 会不断地重启该 Init 容器直到该容器成功为止。 然而，如果 Pod 对应的 restartPolicy 值为 "Never"，并且 Pod 的 Init 容器失败， 则 Kubernetes 会将整个 Pod 状态设置为失败。如果 Pod 的 restartPolicy 设置为 "Always"，Init 容器失败时会使用 restartPolicy 的 "OnFailure" 策略。

如果 Pod 重启，所有 Init 容器必须重新执行。因为 Init 容器可能会被重启、重试或者重新执行，所以 Init 容器的代码应该是幂等的。 特别地，基于 emptyDirs 写文件的代码，应该对输出文件可能已经存在做好准备。

该类容器的作用是做一些初始化的工作,  所以Init容器有如下特点:

* Init容器总是会运行完成
* Init容器按照顺序去运行
* 如果有多个Init容器,  Init容器会在下一个Init容器启动之前完成

#### Init容器与普通容器的不同之处
Init 容器支持应用容器的全部字段和特性，包括资源限制、数据卷和安全设置。 然而，Init 容器对资源请求和限制的处理稍有不同.

同时 Init 容器不支持 lifecycle、livenessProbe、readinessProbe 和 startupProbe， 因为它们必须在 Pod 就绪之前运行完成。







