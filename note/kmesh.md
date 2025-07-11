## controller
与istiod建立连接，接受从istiod过来的配置信息，也就是说kmesh是通过istio来进行配置的。
`istiod`：负责将控制平面的配置分发给数据平面的envoy代理。
1. controller会从istiod那里获取所有的workload和service的地址。构建起从address到workload和service结构体的map。以waypoint和service的联系。这里可以看[k8s.md](./k8s.md#envoy)中的xDS部分。
2. 更新有关的Cache。
3. 同步更新 BPF map。

## workload 
在kmesh中，workload是一个具体的，可寻址的工作负载实例，是数据平面路由的基本单元。
对于kubernetes中的资源其可以是：
1. pod: 最常见的情况
2. deployment 创建的pod
3. statefulset 创建的pod
4. daemonset 创建的pod
5. job/cronjob 创建的pod

非kubernetes资源
1. VM虚拟机
2. 物理机上的服务
3. 外部服务端点
4. Gateway 后面的服务

## service 
**endpoint**: 是 service - workload 之间的映射关系，每一个endpoint都有自己的优先级。endpoint有两个核心的数据结构，EndpointKey 和 EndpointValue。
在 EndpointValue中backenduid是某一个workload的uid。EndpointKey中存储的是serviceid，这样就得到了 serviceid 到 backendid 的映射关系，也就得到了service到workload的映射关系。
``` go
type EndpointKey struct {
    ServiceId    uint32  // 服务ID
    Prio         uint32  // 优先级 (0-6)
    BackendIndex uint32  // 该优先级组内的索引
}

type EndpointValue struct {
    BackendUid uint32    // 对应的 workload ID
}
```

## go 中四张 map
包含serviceid、没有优先级的Endpoint数量，负载均衡策略，服务端口，目标端口，waypoint地址和端口。
客户端访问一个workload的顺序是：FrontendMap -> ServiceMap -> EndpointMap -> BackendMap。通过backendvalue中的workload ip地址和端口来访问workload。
``` go
// Key: BackendKey
type BackendKey struct {
    BackendUid uint32  // workload ID
}

// Value: BackendValue
type BackendValue struct {
    Ip           [16]byte       // workload IP 地址
    ServiceCount uint32         // 该 workload 属于多少个服务
    Services     [10]uint32     // 该 workload 属于的服务列表
    WaypointAddr [16]byte       // Waypoint 地址
    WaypointPort uint32         // Waypoint 端口
}

// Key: EndpointKey
type EndpointKey struct {
    ServiceId    uint32  // 服务ID
    Prio         uint32  // 优先级 (0-6)
    BackendIndex uint32  // 该优先级组内的索引
}

// Value: EndpointValue  
type EndpointValue struct {
    BackendUid uint32    // 对应的 workload ID
}

// Key: ServiceKey
type ServiceKey struct {
    ServiceId uint32
}

// Value: ServiceValue
type ServiceValue struct {
    EndpointCount [7]uint32      // 每个优先级组的 endpoint 数量
    LbPolicy      uint32         // 负载均衡策略
    ServicePort   [10]uint32     // 服务端口
    TargetPort    [10]uint32     // 目标端口
    WaypointAddr  [16]byte       // Waypoint 地址
    WaypointPort  uint32         // Waypoint 端口
}

// Key: FrontendKey
type FrontendKey struct {
    Ip [16]byte  // IP 地址
}

// Value: FrontendValue
type FrontendValue struct {
    UpstreamId uint32  // 对应的服务ID或workload ID
}
```
handleService -> updateServiceMap -> updateEndpointPriority -> updateEndpointOneByOne 这个函数构建起serviceid到endpoint的映射关系。同时构建EndpointKey和EndpointValue的信息。
在endpointCache中构建起 \[endpoint][workloadUid][endpoint]的映射关系。
``` go
func updateEndpointOneByOne
```



## BPF
### BPF map
bpf 程序中维护了多张表，其中四张比较核心的是 frontend、service、endpoint 和 backend map。

**如何通过在用户态的go程序访问和更新内核态的BPF map？**
在go程序中，使用四个对应的cache来记录四张表的内容，这四张表会和内核态的BPF map进行绑定和同步。所以这里涉及到go程序中的四个cache和内核态的BPF map是怎么进行绑定和更新的。
1. 需要使用 cilium/ebpf 库来绑定和操作 BPF map。
2. 操作系统会提供对这四张表的访问接口，会在 /sys/fs/bpf/kmesh/ 提供四张表的文件，并提供用于读写的系统调用。
3. 在go程序中构建与 BPF map 对应的结构体。在  workload_controller 中有Processor 结构体，其中含有下述代码片段：

``` go
type Processor struct {
    // ... other fields
    bpf           *bpf.Cache
    // ... other fields
}
```

并且bpf.Cache 中还有一个关键的结构体 `KmeshCgroupSockWorkloadMaps`

``` go
type Cache struct {
    // ... other fields
    bpfMap bpf2go.KmeshCgroupSockWorkloadMaps
    // ... other fields
}
```
KmeshCgroupSockWorkloadMaps 中通过field labels 来绑定内核态中使用的数据，这个过程应当会用到go中的反射机制，这里应该需要看 cilium/ebpf 的有关代码的实现才能知道具体是怎么操作的：
``` go
type KmeshCgroupSockWorkloadMaps struct {
	KmAuthReq     *ebpf.Map `ebpf:"km_auth_req"`
	KmAuthRes     *ebpf.Map `ebpf:"km_auth_res"`
	KmBackend     *ebpf.Map `ebpf:"km_backend"`
	KmCgrTailcall *ebpf.Map `ebpf:"km_cgr_tailcall"`
	KmEndpoint    *ebpf.Map `ebpf:"km_endpoint"`
	KmFrontend    *ebpf.Map `ebpf:"km_frontend"`
	KmLogEvent    *ebpf.Map `ebpf:"km_log_event"`
	KmManage      *ebpf.Map `ebpf:"km_manage"`
	KmPerfInfo    *ebpf.Map `ebpf:"km_perf_info"`
	KmPerfMap     *ebpf.Map `ebpf:"km_perf_map"`
	KmService     *ebpf.Map `ebpf:"km_service"`
	KmSockstorage *ebpf.Map `ebpf:"km_sockstorage"`
	KmTcpProbe    *ebpf.Map `ebpf:"km_tcp_probe"`
	KmTmpbuf      *ebpf.Map `ebpf:"km_tmpbuf"`
	KmWlpolicy    *ebpf.Map `ebpf:"km_wlpolicy"`
	KmXdpTailcall *ebpf.Map `ebpf:"km_xdp_tailcall"`
	KmeshMap1600  *ebpf.Map `ebpf:"kmesh_map1600"`
	KmeshMap192   *ebpf.Map `ebpf:"kmesh_map192"`
	KmeshMap296   *ebpf.Map `ebpf:"kmesh_map296"`
	KmeshMap64    *ebpf.Map `ebpf:"kmesh_map64"`
}
```

## 负载均衡策略
三种负载均衡策略：**Random**、**Strict**、**Failover**。
这里需要注意的是在 bpf 程序的实现中，随机方法和严格方法在代码上都是使用的最高优先级0中随机的bakend_index。我的理解是之所以都是使用最高优先级0，是因为在 random 使用方便，strict 可能则是因为优先级0代表的是本地性，所以严格方法也使用了优先级0， 而优先级的计算则由用户态的go程序来处理。
**Failover** 在 bpf 程序中则是使用了多优先级递减的方式来处理，也就是先在优先级0中随机选择一个endpoint，如果没有可用的endpoint，则递减到优先级1，依次类推，直到找到一个可用的endpoint。
有关代码实现见 *kmesh/bpf/kmesh/workload/include/service.h*

| 策略 | 优先级使用 | 故障处理 | 特点 |
|------|------------|----------|------|
| **Random** | 只用优先级0 | 无故障转移 | 简单随机 |
| **Strict** | 只用优先级0 | 严格失败 | 严格本地性的简单随机 |
| **Failover** | 多优先级递减 | 自动降级 | 容错性强 |


## 语言相关
**特殊的函数指针**
``` c
static void *(*bpf_map_lookup_elem)(void *map, const void *key) = (void *) 1;
```
这个函数指针是一个特殊的函数指针，并不是真正的函数实现，而是一个占位符，在编译时会被替换为实际的系统调用。 (void *) 1 对应辅助函数的id。



#### backend map
#### endpoint map 
[见service部分](#service)

四张表是层层映射的关系

## RBAC 权限模型
三个基本概念：**用户**、**角色**、**权限**
#### RBAC 模型分类
**RBAC0**：最基本的模型，用户和角色之间是多对多的关系，角色和权限之间也是多对多的关系。
**RBAC1**：建立在RBAC0基础上，角色可以分成几个等级，在角色中可以有继承关系。
**RBAC2**：建立在RBAC0基础上，对用户、角色和权限三者之间增加了限制，静态职责分离和动态职责分离。
**RBAC3**：RBAC1 + RBAC2 的结合。
