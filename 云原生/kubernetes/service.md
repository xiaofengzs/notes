# Service
## 为什么需要Service
在Kubernetes中,  Pod可能被随时杀死,  或者创建一个新的.  在这种情况下,  如果想要访问某一个服务的Pod,  必须得不断的去重新获取Pod的IP地址,  这样会很麻烦.  
Service是用来解决这个问题的,  Service提供一个稳定的IP地址并且能够转发请求给Pod.  所以想要访问某个服务,  可以直接访问这个服务的Service,  由Service来转发请求.  即使Pod的数量增加或者减少,  都由Service来处理这种变化.

## 什么是Service
Service是一种为一组功能相同的Pod提供单一不变的接入点的资源,  当服务存在时,  它的IP地址和端口不会改变.

## Service的三种使用方式
### 服务发现
创建Service,  可以通过一个单一稳定的IP地址访问Pod.  在服务整个生命周期内地址保持不变.  再服务后面的Pod可能删除重建,  Pod的IP地址可能会改变,  数量可能会增减,  但是始终可以通过Service访问到这些Pod.
#### 通过环境变量访问Service
在Pod开始运行的时候,  kubernetes会初始化一系列的环境变量指向现在存在的服务.  如果创建的Service早于客户端Pod的创建,  Pod上的进程可以根据环境变量获取Service的IP地址和端口号.
```
kubectl exec <pod-name> env -n <namespace>
```
#### 通过DNS访问Service
在kube-system命名空间中, 有运行成DNS服务. 在集群中的其他Pod都被配置成使用其作为dns.  运行在pod上的进程DNS查询都会被kubernetes自身的DNS服务器响应,  改服务器知道系统中运行的所有服务.  
每个服务从内部DNS服务器中获得一个DNS条目,  客户端的pod在知道服务名称的情况下可以通过全限定域名(FQDN)来访问,  而不是用环境变量.

一个FQND的例子, `backend-database.default.svc.cluster.local`. 通过这个DQDN可以来访问某个服务.  backend-database对应于服务名称,  default表示服务在其中定义的名称空间,  而svc.cluster.local是在所有服务名称中使用的可配置集群域后缀.

如果两个服务在同一个namespace下,  那么只需要通过服务名来访问就可以了`backend-database`.

使用FQDN访问服务,  如果被访问的端口不是标准端口号,  必须指定访问的端口号.
### 代理外部集群的应用程序
在kubernetes中,  有时想要从集群中访问集群外的服务,  这时可以使用Service来代理集群外的服务.  这样可以充分利用Service的服务发现和负载平衡的功能.  
使用这个功能,  我们在Service中不使用selector,  而是自己创建与Service名字相同的Endpoint,  这样Service会自动绑定Endpoint.


### 将集群内的服务暴露给外部客户端
有时候,  我们想要把服务暴露给外部客户端访问.  Service提供了三种类型来实现这个目标.
Service有如下几种类型:  
1.  ClusterIP: 在集群内部 IP 上公开服务。 选择此值使服务只能从集群内访问。 这是默认的服务类型。  
2. NodePort:  在Node节点上开放一个静态端口来暴露Service服务. 这种方式是通过在Node上开放一个静态端口,  这个端口的流量会被转发到一个ClusterIP的service上. 可以通过请求 <NodeIP>:<NodePort> 从集群外部访问 NodePort 服务。
3. LoadBalancer:  是NodePort类型的一种扩展, 使用云提供商的负载均衡器在外部公开服务。 外部负载均衡器把路由重定向到跨节点的端口上.
4. ExternalName:  
