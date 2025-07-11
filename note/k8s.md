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

在每一个node上都会有一个CoreDNS作为DaemonSet运行，接受来自pod的DNS解析请求。


#### 负载均衡机制实现原理
1. **iptables**
2. **ipvs**。service上分配的clusterIP实际上会新建一个虚拟网口，并将service的clusterIP和端口绑定到这个虚拟网口上。容器或者节点上进程访问地址的时候，会进入到ipvs的input hook点，然后进行负载均衡。
3. **eBPF**。


## Envoy
边缘和服务代理。作为服务代理软件，其可以扮演两种角色：1. 集群流量入口API网管 2. 作为服务sidecar拦截并治理服务网格中的东西流量。
**基础概念**
- "Listener"：Envoy打开的监听端口，用于接收Downstream（客户端）连接。Listener对应的配置/资源发现服务称之为`LDS`。
- "Cluster"：Envoy连接的上游服务都被抽象为一个Cluster。Cluster对应的配置/资源发现服务称之为`CDS`。当Cluster类型为`EDS`时，说明该Cluster的所有endpoints都需要由xDS服务下发，而不是用DNS等方式解析。下发endpoints的服务称之为`EDS`。
- "Router"：Listener可以接受来自下游的连接，Cluster可以将流量发送给具体的上游服务，而Router则决定在接收到下游的连接和数据之后，应该将数据交给哪一个Cluster处理。定义了数据的分发规则。Router对应的配置/资源发现服务称之为`RDS`。
- "Filter"：通俗的讲就是插件，提供了强大的可扩展能力。Filter所有的配置都是嵌入在LDS、CDS和RDS中的。
`xDS` 是 Envoy 的配置和服务发现机制的统称，包含了上述的 LDS、CDS、EDS 和 RDS。Envoy 通过 xDS 接口来获取和更新其运行时配置。

### 增量xDS
Envoy控制面下发的discoveryResponse是一个全量的配置，包含了所有的资源。造成大量不必要的流量开销。另外，由于sidecar并不知道网格中服务需要访问哪些服务，所以会记录网格中所有服务的相关信息，而一个服务往往只需要访问少数几个服务。
在delta xDS中，Envoy会在每次更新时只发送变更的资源。
**基于 delta xDS 实现三种新功能**
1. **Lazy Loading**：在实际的配置被使用的时候再订阅该资源，从控制面获取相关配置，首次访问性能会受到一定的影响。
2. **Incremental Updates**：当部分资源发生变化时，Envoy只会更新变更的资源，而不是全量更新。这样可以减少不必要的流量开销。
3. **缓存擦除**：根据当前负载实际请求情况动态调整订阅资源类型，对于不再活跃的配置，取消订阅，从Envoy内存中擦除。在超大规模场景中减少内存占用。