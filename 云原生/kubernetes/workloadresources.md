## Controller
### 什么是Controller
 在集群上管理和运行容器的对象
### Pod和Controller关系
Pod是通过Controller实现应用的运维,  比如伸缩,  滚动升级等等.

Pod和Controller之间通过label标签建立关系.
## ReplicaSet
### ReplicaSet是为了解决什么问题
运行在Pod中的服务,  如果出现了不可恢复的问题,  就会挂掉或者宕机.  如果发生这类错误时,  都需要使用人工来重启服务的话,  效率太低下.  Kubernetes提供了一种功能,  可以把Pod的数量保持在期望值,  如果有崩溃的错误发生,  可以重启Pod来恢复服务. K8s提供了两种资源类型, ReplicaSet和ReplicaController.
### ReplicaSet是什么
ReplicaSet可以维持一组在任何时间运行的稳定数量的副本Pod. 因此,  ReplicaSet可以用来保证一组Pod的可用性. 即如果有容器异常退出，会自动创建新的 Pod 来替代；而如果异常多出来的容器也会自动回收。

在新版本的 Kubernetes 中建议使用 ReplicaSet 来取代 ReplicationController。ReplicaSet 跟 ReplicationController 没有本质的不同，只是名字不一样，并且 ReplicaSet 支持集合式的 selector。

虽然 ReplicaSet 可以独立使用，但一般还是建议使用 Deployment 来自动管理 ReplicaSet，这样就无需担心跟其他机制的不兼容问题（比如 ReplicaSet 不支持 rolling-update 但 Deployment 支持）。
## Deployment
### deployment是什么
Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义（declarative）方法，用来替代以前的 ReplicationController 来方便的管理应用。典型的应用场景包括：

- 定义 Deployment 来创建 Pod 和 ReplicaSet
- 滚动升级和回滚应用
- 扩容和缩容
- 暂停和继续 Deployment

## SatefulSet
### SatefulSet是什么
StatefulSet 是用于管理有状态应用的工作负载对象，与 ReplicaSet 和 Deployment 这两个对象不同，StatefulSet 不仅能管理 Pod 的对象，还它能够保证这些 Pod 的顺序性和唯一性。

## DaemonSet
DaemonSet是这样一种对象（守护进程），它在集群的每个节点上运行一个Pod，且保证只有一个Pod，这非常适合一些系统层面的应用，例如日志收集、资源监控等，这类应用需要每个节点都运行，且不需要太多实例，一个比较好的例子就是Kubernetes的kube-proxy。   
DaemonSet跟节点相关，如果节点异常，也不会在其他节点重新创建。

## Job
Job 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

## CronJob
Cron Job 管理基于时间的 Job，即：

- 在给定时间点只运行一次
- 周期性地在给定时间点运行

一个 CronJob 对象类似于 crontab （cron table）文件中的一行。它根据指定的预定计划周期性地运行一个 Job，格式可以参考 Cron 。