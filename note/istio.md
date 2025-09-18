# Istio


## sidecar
通过sidecar的好处
1. 解耦与统一管理
2. 网络策略与流量控制
3. 安全性增强
4. 可观测性
5. 多语言异构环境
5. 可扩展的灵活性

**如何实现代理**
1. 透明代理：sidecar和应用容器在同一个pod中，属于同一个网络命名空间。sidecar通过iptables规则拦截应用容器的流量，将流量转发到sidecar容器中，然后由sidecar容器处理。
2. 对于出站流量会使用iptables将应用容器的流量重定向到sidecar容器中。所以目的ip和源ip都是sidecar的ip地址。


### Envoy
边缘和服务代理。作为服务代理软件，其可以扮演两种角色：1. 集群流量入口API网管 2. 作为服务sidecar拦截并治理服务网格中的东西流量。
**基础概念**
- "Listener"：Envoy打开的监听端口，用于接收Downstream（客户端）连接。Listener对应的配置/资源发现服务称之为`LDS`。
- "Cluster"：Envoy连接的上游服务都被抽象为一个Cluster。Cluster对应的配置/资源发现服务称之为`CDS`。当Cluster类型为`EDS`时，说明该Cluster的所有endpoints都需要由xDS服务下发，而不是用DNS等方式解析。下发endpoints的服务称之为`EDS`。
- "Router"：Listener可以接受来自下游的连接，Cluster可以将流量发送给具体的上游服务，而Router则决定在接收到下游的连接和数据之后，应该将数据交给哪一个Cluster处理。定义了数据的分发规则。Router对应的配置/资源发现服务称之为`RDS`。
- "Filter"：通俗的讲就是插件，提供了强大的可扩展能力。Filter所有的配置都是嵌入在LDS、CDS和RDS中的。
`xDS` 是 Envoy 的配置和服务发现机制的统称，包含了上述的 LDS、CDS、EDS 和 RDS。Envoy 通过 xDS 接口来获取和更新其运行时配置。

#### Envoy 处理链

envoy 的处理过程是通过一系列的过滤器（Filters）组成的过滤器链（Filter Chain）来实现的。

**核心概念**

- **Listener**：监听器，负责监听特定的网络端口。sidecar 默认有两个 listener，一个用于处理入站流量（inbound listener），另一个用于处理出站流量（outbound listener）。
- **Filter**：过滤器，处理链中的基本单元，负责完成具体的任务，如协议解析、路由选择、认证等。
    - **Network Filters**：处理网络层的流量，如 TCP 连接管理、TLS 终止等。
    - **HTTP Filters**：处理 HTTP 层的流量，如HTTP 路由、请求/响应修改等。
- **Filter Chain**：过滤器链，一个 listener 可以包含多个过滤器链，每个链可以包含多个过滤器。envoy会根据连接的目标端口、IP地址、SNI等信息来选择合适的过滤器链进行处理。

#### 增量xDS
Envoy控制面下发的discoveryResponse是一个全量的配置，包含了所有的资源。造成大量不必要的流量开销。另外，由于sidecar并不知道网格中服务需要访问哪些服务，所以会记录网格中所有服务的相关信息，而一个服务往往只需要访问少数几个服务。
在delta xDS中，Envoy会在每次更新时只发送变更的资源。
**基于 delta xDS 实现三种新功能**
1. **Lazy Loading**：在实际的配置被使用的时候再订阅该资源，从控制面获取相关配置，首次访问性能会受到一定的影响。
2. **Incremental Updates**：当部分资源发生变化时，Envoy只会更新变更的资源，而不是全量更新。这样可以减少不必要的流量开销。
3. **缓存擦除**：根据当前负载实际请求情况动态调整订阅资源类型，对于不再活跃的配置，取消订阅，从Envoy内存中擦除。在超大规模场景中减少内存占用。

## 其他
## RBAC 权限模型
三个基本概念：**用户**、**角色**、**权限**
#### RBAC 模型分类
**RBAC0**：最基本的模型，用户和角色之间是多对多的关系，角色和权限之间也是多对多的关系。
**RBAC1**：建立在RBAC0基础上，角色可以分成几个等级，在角色中可以有继承关系。
**RBAC2**：建立在RBAC0基础上，对用户、角色和权限三者之间增加了限制，静态职责分离和动态职责分离。
**RBAC3**：RBAC1 + RBAC2 的结合。

## ambient

### waypoint
用于L7代理。关于waypoint istio的官方文档进行了详细的介绍。见 [Istio Waypoint](https://istio.io/latest/zh/docs/ambient/usage/waypoint/)

