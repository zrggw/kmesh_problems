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

### BPF 程序
首先是内核传入的bpf_sock_addr结构体，然后根据bpf_sock_addr结构体构造出kmesh_ctx结构体。
核心函数为cgroup_connect4_prog和cgroup_connect6_prog，这两个函数会被内核调用，传入的参数为bpf_sock_addr结构体。
见 *kmesh/bpf/kmesh/workload/cgroup_sock.c*.


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

## IPsec
参阅 [IPsec](https://www.zhuanlan.zhihu.com/p/44874772)。[Linux XFRM Reference Guide for IPsec](https://pchaigno.github.io/xfrm/2024/10/30/linux-xfrm-ipsec-reference-guide.html)。
**kmesh 中的 IPsec**，见项目的kmesh/docs/proposal/kmesh_support_encrypt.md文件。

**在这里感觉需要补充一些关于Linux内核网络的内容，搞清楚具体的流程，IPSec和eBPF分别是在内核流程中的哪一部分发挥作用。见[linux.md](linux.md#网络)**

IPsec 是一种网络安全协议，用于提供IP层的安全性。IPsec提供了两种安全机制**认证**和**加密**。
IPsec还需要有密钥的管理和交换功能。
IPsec的两种工作模式：
1. **传输模式**：只对IP数据包的有效载荷进行加密和认证，IP头部不变。
2. **隧道模式**：对整个IP数据包进行加密和认证，IP头部被替换为新的IP头部。

IPSec的的设置是单向的。

### IPsec协议组
IPsec协议组包括：
1. **AH**（Authentication Header）：提供数据包的认证和完整性保护，但不提供加密。可以单独使用，也可以在隧道模式下，或与ESP一起使用。
2. **ESP**（Encapsulating Security Payload）：定义了加密和和可选认证的应用方法。由三部分组成：ESP头部、ESP有效载荷和ESP可选尾部。头部包含两部分：安全策略索引和序列号。ESP有效载荷是加密后的数据包内容。尾部包含填充和认证数据。
3. **IKE**（Internet Key Exchange）：主要是对密钥交换进行管理，主要包括三个功能：
    - **密钥协商**：对使用的协议、就爱密算法和密钥进行协商
    - **密钥交换**：方便的密钥交换机制
    - **跟踪**： 对以上的这些约定进行跟踪。

### IPsec的工作模式
IPsec的工作模式有两种：**传输模式**和**隧道模式**。
1. **传输模式**：只对IP数据包的有效载荷进行加密和认证，IP头部不变。
2. **隧道模式**：对整个IP数据包进行加密和认证，IP头部被替换为新的IP头部。

#### IPSec 的数据包传输流程
数据包解密的过程不依赖于Policy，首先是找到对应的State进行解密。


### Kmesh中的IPsec
#### 设计文档
见 *kmesh/docs/proposal/kmesh_support_encrypt.md*。
在每个容器的出口网卡上添加一个tc(traffic control)程序。将pod中发出的流量打上**mark**标签，标记为走ipsec加密的流量。

#### 数据结构和接口
**KmeshNodeInfo**：KmeshNodeInfo CRD 定义。记录每个node的kmesh信息，包括node的ip地址、bootid、spi版本号等。

**IPsec handler**：管理kmesh中的IPsec密钥和Linux XFRM规则。

**State 和 Policy**：
- `State`：定义了Linux内核XFRM框架中的核心数据结构，用于定义IPSec加密连接的状态信息，描述了如何对网络数据包进行加密/解密处理。
- `Policy`：定义了什么流量需要加密处理，流量匹配条件、处理方向。
``` go
state := &netlink.XfrmState{
    Src:   src,                          // 隧道源IP
    Dst:   dst,                          // 隧道目标IP
    Proto: netlink.XFRM_PROTO_ESP,       // 使用ESP协议
    Mode:  netlink.XFRM_MODE_TUNNEL,     // 隧道模式
    Spi:   ipsecKey.Spi,                 // 安全参数索引
    Reqid: 1,                            // 请求ID (用于关联Policy)
    Aead: &netlink.XfrmStateAlgo{        // 加密算法配置
        Name:   ipsecKey.AeadKeyName,    // 算法名称
        Key:    key,                     // 加密密钥
        ICVLen: ipsecKey.Length,         // 完整性检查长度
    },
}

policy := &netlink.XfrmPolicy{
    Src: srcCIDR,                        // 源网络范围 (如 0.0.0.0/0)
    Dst: dstCIDR,                        // 目标网络范围 (如 Pod CIDR)
    Dir: netlink.XFRM_DIR_IN/OUT/FWD,    // 流量方向
    Tmpls: []netlink.XfrmPolicyTmpl{     // 使用的模板
        {
            Src:   src,                  // 隧道源IP
            Dst:   dst,                  // 隧道目标IP
            Proto: netlink.XFRM_PROTO_ESP,
            Reqid: 1,                    // 关联到对应的State
            Mode:  netlink.XFRM_MODE_TUNNEL,
        },
    },
    Mark: &netlink.XfrmMark{            // 流量标记
        Value: mark,                     // 0xd0 (入站) 或 0xe0 (出站)
        Mask:  0xffffffff,
    },
}
```

createXfrmRuleIngress 创建入站规则的时候使用的是网卡的IP地址。在生成密钥的时候使用的也是网卡的IP地址。
createStateRule 根据src dst网卡地址构建state规则。
createPolicyRule，传入的src参数是一个常亮也就是0.0.0.0地址，dst是pod的地址

ingress mark := uint32(0xd0) 
egress mark := uint32(0xe0)


主要数据结构
``` go
type IpSecKey struct {
    Spi         int    `json:"spi"`        // Security Parameter Index，IPSec 安全参数索引
    AeadKeyName string `json:"aeadKeyName"` // AEAD 算法名称（如 rfc4106）
    AeadKey     []byte `json:"aeadKey"`     // AEAD 加密密钥
    Length      int    `json:"length"`      // 密钥长度
}

type IpSecHandler struct {
    Spi             int                    // 当前使用的 SPI
    mutex           sync.RWMutex           // 读写锁，保证并发安全
    watcher         filewatcher.FileWatcher // 文件监控器
    historyIpSecKey map[int]IpSecKey       // 历史 IPSec 密钥缓存
}
```

核心功能模块：
1. 密钥管理 & 文件监控
- `LoadIPsecKeyFromFile`：从文件加载IPsec密钥。`LoadIPsecKeyFromIO`：从IO读取IPsec密钥。
- `StartWatch`：启动文件监控，监听密钥文件时重新加载。

2. 密钥生成
- `generateIPSecKey`：基于源IP、目标IP、源BootID、目标BootID和原始密钥，使用SHA512生成新的IPSec密钥。

3. XFRM规则管理

入站规则：
- `createXfrmRuleIngress`,创建入站IPSec规则。
- `State规则`：定义ESP协议的加密状态
- `Policy规则`：定义流量策略(IN 和 FWD 方向)

出站规则：
- `createXfrmRuleEgress`：创建出站IPSec规则。
- Policy 规则：定义流量策略(OUT 方向)

#### 控制平面逻辑

#### 数据平面实现

#### 用户接口