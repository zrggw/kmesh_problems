## list and watch
### informer
Informer是Kubernetes的一个机制，用于监听和缓存Kubernetes API服务器中的资源对象变化。它通过watch机制实时获取资源对象的变化，并将其缓存到本地内存中，以提高访问速度和减少API服务器的负载。

### lister
Lister是Kubernetes的一个机制，用于从缓存中获取资源对象。它提供了一种高效的方式来列出和获取特定资源类型的对象，而无需直接访问API服务器。Lister通常与Informer配合使用，以提高性能和减少API调用。

### service 
service 提供稳定的ip地址和dns以及负载均衡。但是其实际上是依赖于kube-proxy的。

**pod之间通信过程举例**
1. pod中的进程首先在开发时约定了service的名称或者是环境变量中约定或者在configmap中约定。因此得到URL。
2. 进程发起请求之后会由系统进行DNS解析，得到service的clusterIP地址。
3. 内核会查询路由表，发现需要走默认网关。
4. kube-proxy会创建iptables规则拦截数据包。
5. iptables规则会将目标地址转换为实际的pod IP地址。
6. 内核会将数据包路由到实际的pod。

**service 与 endpoint 之间的关系**
Endpoint controller通过api servser的的watch机制来实时感知pod的变化。如果出现了变化，Endpoint controller会获取pod列表，筛选出属于该service的pod，提取出pod的ip地址和端口，将这些信息存储到Endpoint对象中。

**service 与 kube-proxy 之间的关系**
Service 提供稳定的虚拟 IP 和服务发现，kube-proxy 负责将这个虚拟 IP 的流量实际转发到后端 Pod，两者配合实现了 Kubernetes 的服务网络。

**dns 解析过程**
k8s会提供一个dns解析服务，并在pod文件系统中的配置文件（操作系统读取 /etc/resolv.conf）中指定dns解析服务的地址，然后由操作系统将dns解析的请求发送往dns解析服务。所以在应用开发的时候就需要使用url的形式来访问服务，而不是直接使用ip地址，并读取环境变量或者配置文件中的服务地址。

在每一个node上都会有一个CoreDNS作为DaemonSet运行，接收来自pod的DNS解析请求。


#### 负载均衡机制实现原理
1. **iptables**
2. **ipvs**。service上分配的clusterIP实际上会新建一个虚拟网口，并将service的clusterIP和端口绑定到这个虚拟网口上。容器或者节点上进程访问地址的时候，会进入到ipvs的input hook点，然后进行负载均衡。
3. **eBPF**。

### CRD (Custom Resource Definition)
见 [k8s官方文档](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

CRD是Kubernetes的扩展机制，允许用户定义自己的资源类型。同样通过yaml文件进行定义和创建，通过kubectl命令进行管理。
两部分组成：**CRD定义**和**CR实例**。
CRD定义了资源的名称、版本、规范等信息，而CR实例则是该资源类型的具体实例。还需要一个**Controller**来管理CR实例的生命周期。
