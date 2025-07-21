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

#### KNIMap
KNIMap 是一个 LPM Trie 类型的 BPF map，用于存储 Kmesh 管理的 CIDR 信息。
其中，key是kmesh节点中pod的CIDR地址，value是一个简单的标记值（uint32），表示该 CIDR 由 Kmesh 管理，如果是kmesh管理的value为1。

在tc的bpf程序中会检查这个表，判断是否是kmesh管理的CIDR地址，如果是，则将数据包打上mark，表示需要进行IPSec加密处理。



### BPF 程序
首先是内核传入的bpf_sock_addr结构体，然后根据bpf_sock_addr结构体构造出kmesh_ctx结构体。
核心函数为cgroup_connect4_prog和cgroup_connect6_prog，这两个函数会被内核调用，传入的参数为bpf_sock_addr结构体。
见 *kmesh/bpf/kmesh/workload/cgroup_sock.c*.

**cgroup_connect4_prog 和 cgroup_connect6_prog何时被触发**：
已知的是bpf程序会在进入系统调用之前被触发，那么这两个函数应该是会在调用 connect 系统调用时被触发。因此此时应该还未进入到内核的网络栈中。所以我的理解是bpf程序是工作在内核的网络栈之前的。

#### cgroup_connect4_prog
如果目标的ip地址还没有被kmesh接管，那么就会进行查表。
1. 首先会从kmesh_ctx中获取用户程序想要连接的目标地址，构建出 frontend_key，ip地址为用户程序想要连接的目标地址。
2. 然后会去查找 frontend_map，得到对应的 frontend_value。其中包含了目标的serviceid或者workloadid，即upstream_id。
3. 然后接着查找 service_map，得到对应的 service_value。
4. 接着会查找 endpoint_map，得到对应的 endpoint_key和endpoint_value。
5. 然后会查找 backend_map，得到对应的 backend_key和backend_value。
6. 最后会将目标地址修改为 backend_value 中的 ip 地址和端口。

上述是bpf程序的主要逻辑，还有些其他部分没有搞清楚。如handle_kmesh_manage_process和observe_on_pre_connect等函数的具体功能是什么，目前只知道也会进行查表操作。
在经过bpf程序之后，数据应该会进入内核的网络栈中。

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


### BPF中的IPsec
在tc_mark_encrypt函数中会将数据包的mark设置为0xe0，表示需要进行IPsec加密处理。



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

## ipsec

关于IPsec的内容，见[linux.md](linux.md#IPsec)。

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

**kmesh中如何设置State和Policy规则？**
- `CreateXfrmRule`：创建入站和出站的IPSec规则。
    - `createXfrmRuleIngress`：在其中调用`createStateRule`和`createPolicyRule`来创建入站的IPSec规则。
        - `createStateRule`：根据src和dst网卡地址构建State规则。src为远程node的网卡地址，dst为本地node的网卡地址。
        - `createPolicyRule`：src设置为0.0.0.0，dst设置为本地pod的CIDR地址，tmpl中src设置为远程node的网卡地址，dst设置为本地node的网卡地址。

- `createXfrmRuleEgress`：创建出站的IPSec规则。
    - `createStateRule`：根据src和dst网卡地址构建State规则。src为本地node的网卡地址，dst为远程node的网卡地址。
    - `createPolicyRule`：src设置为0.0.0.0, dst设置为远程pod的CIDR地址，tmpl中src设置为本地node的网卡地址，dst设置为远程node的网卡地址。

- `createStateRule`：根据src和dst网卡地址构建State规则。
- `createPolicyRule`：传入的srcCIDR, dstCIDR, tmpl中的src和dst地址。

``` go
state := &netlink.XfrmState{
		Src:   src,
		Dst:   dst,
		Proto: netlink.XFRM_PROTO_ESP,
		Mode:  netlink.XFRM_MODE_TUNNEL,
		Spi:   ipsecKey.Spi,
		Reqid: 1,
		Aead: &netlink.XfrmStateAlgo{
			Name:   ipsecKey.AeadKeyName,
			Key:    key,
			ICVLen: ipsecKey.Length,
		},
	}

policy := &netlink.XfrmPolicy{
		Src: srcCIDR,
		Dst: dstCIDR,
		Tmpls: []netlink.XfrmPolicyTmpl{
			{
				Src:   src,
				Dst:   dst,
				Proto: netlink.XFRM_PROTO_ESP,
				Reqid: 1,
				Mode:  netlink.XFRM_MODE_TUNNEL,
			},
		},
		Mark: &netlink.XfrmMark{
			Mask: 0xffffffff,
		},
	}
```



#### 控制平面逻辑

**IPsec Controller**
- NewIPsecController：首先会构建一个k8s的clientset，然后创建一个KmeshNodeInfo的Lister和Informer。接着创建一个IPSecController实例。然后加载IPsec密钥文件。然后通过k8s的clientset获取kmesh节点信息。
有了本地节点信息之后，会将本地节点信息添加到IPSecController的kmeshNodeInfo中，见下面代码。localNodeName是从环境变量中获取的，表示当前节点的名称。然后提取本地节点的NodeInternalIP地址，并将其添加到kmeshNodeInfo的Addresses中。然后注册事件处理器，监听集群节点变化。

- 什么是informer和lister？
Informer 是 Kubernetes 中的一个组件，可以监听 Kubernetes API Server 上的资源的变化事件。
Lister 是一个查询工具，可以从 Kubernetes API Server 中获取资源列表，或者从本地缓存中读取资源信息。
一个是监听资源变化，一个是查询资源信息。

- Run：启动Informer，等待缓存同步完成。然后调用attachTcDecrypt函数来附加TC解密程序，在本地节点的网络接口上附加bpf tc解密程序。接着调用syncAllNodeInfo函数来同步所有节点信息。然后更新本地的kmesh节点信息，通知其他节点更新密钥。然后启动IPSec密钥文件的监听。最后开启一个新的goroutine来处理工作队列中的任务。

    - syncAllNodeInfo：在这个函数中，首先获取所有的kmesh节点信息，然后遍历每个节点的信息，调用handleOneNodeInfo函数来处理每个节点的信息。构建IPsec的State和Policy规则。
        - handleOneNodeInfo：处理每一个单独的节点信息。在其中会调用CreateXfrmRule函数来创建入站和出站的IPSec规则。遍历node中每一个pod的ip，调用updateKNIMapCIDR来更新KNIMap中的CIDR信息。

    - attachTcDecrypt：附加TC解密程序到本地节点的网络接口上。这里的网络接口应该是指本地节点的物理网卡。负责在特定的命名空间附加TC解密程序。首先获取本地节点的网络命名空间路径，因为kmesh在容器中运行，需要访问宿主机的网络命名空间。然后使用netns.WithNetNSPath函数来切换到本地节点的网络命名空间，在本地命名空间中执行handleTc函数。

    - handleTc：处理TC解密程序的附加和移除。根据传入的mode参数来决定是附加还是移除TC解密程序。首先获取所有的网络接口，遍历和过滤掉回环接口。然后检查每个接口是否包含本地节点的IP地址，如果包含则继续处理，附加或者分离TC解密程序，通过ManageTCProgram函数来附加或分离TC解密程序。

    - ManageTCProgram：负责具体在网络接口上附加或分离TC解密程序。首先设置队列规则（qdisc），然后创建BpfFilter并附加到ingress钩子点。这里的qdisc是指队列规则，是Linux内核中用于管理网络流量的机制。BpfFilter是指BPF过滤器，用于过滤和处理网络数据包。



关于附加解密程序的逻辑如下：
``` plain
┌─────────────────────────────────────────────────────────────────┐
│                    attachTcDecrypt()                            │
│                                                                 │
│  1. 获取宿主机网络命名空间路径                                    │
│  2. 定义附加函数                                                 │
│  3. 进入宿主机网络命名空间                                        │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                  handleTc(TC_ATTACH)                        ││
│  │                                                             ││
│  │  1. 获取所有网络接口                                         │ │
│  │  2. 过滤回环接口                                             │ │
│  │  3. 检查接口是否包含节点IP                                    │ │
│  │  4. 对匹配的接口附加TC程序                                    │ │
│  │     │                                                       │ │
│  │     ▼                                                       │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │              ManageTCProgram()                          │ │ │
│  │  │                                                         │ │ │
│  │  │  1. 设置队列规则（qdisc）                                │ │ │
│  │  │  2. 创建BpfFilter                                       │ │ │
│  │  │  3. 附加到ingress钩子点                                  │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

如何附加TC解密程序到网络接口 ManageTCProgramByFd 函数中实现：

### TC 框架核心概念解释：

**1. Qdisc（队列规则）**：
- Linux 内核中管理网络流量的机制
- clsact 是专门用于分类和动作处理的特殊 qdisc
- 提供 ingress/egress 钩子点，不进行实际队列调度

**2. BpfFilter（eBPF 过滤器）**：
- 将 eBPF 程序附加到网络接口的桥梁
- 包含程序的文件描述符和配置参数
- DirectAction=true 允许 eBPF 程序直接控制数据包命运

**3. eBPF 程序 FD（文件描述符）**：
- bpf2go 编译 C 代码 → Go 结构体
- LoadAndAssign() 加载到内核 → 返回 FD
- FD 是内核中已加载 eBPF 程序的引用

### 完整的附加流程图：

┌─────────────────────────────────────────────────────────────────┐
│                    网络接口 (eth0)                                │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   clsact qdisc                              │ │
│  │  (专门用于分类和动作处理，不缓存数据包)                         │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │                ingress 钩子点                            │ │ │
│  │  │  (数据包进入网络接口时的处理点)                            │ │ │
│  │  │  ┌─────────────────────────────────────────────────────┐ │ │ │
│  │  │  │              BpfFilter                              │ │ │ │
│  │  │  │  • Name: tc_ingress-eth0                            │ │ │ │
│  │  │  │  • Protocol: ETH_P_ALL (处理所有协议类型)           │ │ │ │
│  │  │  │  • Priority: 1 (过滤器优先级)                      │ │ │ │
│  │  │  │  • DirectAction: true (允许直接返回动作)            │ │ │ │
│  │  │  │  • Fd: eBPF程序文件描述符 (指向内核中的程序)         │ │ │ │
│  │  │  └─────────────────────────────────────────────────────┘ │ │ │
│  │  └─────────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  数据包处理流程:                                                  │
│  网络 ──→ 网络接口 ──→ clsact qdisc ──→ BpfFilter ──→ eBPF程序   │
│                                    ↓                           │
│            eBPF程序通过FD执行tc_mark_decrypt函数                 │
│                                    ↓                           │
│                   返回 TC_ACT_OK ──→ 继续到内核网络栈            │
│                   返回 TC_ACT_DROP ──→ 丢弃数据包               │
└─────────────────────────────────────────────────────────────────┘

### eBPF 程序 FD 的完整来源链路：

┌─────────────────────────────────────────────────────────────────┐
│                   编译时 (bpf2go)                                │
│                                                                 │
│  tc_mark_decrypt.c ──→ bpf2go 工具 ──→ kmeshtcmarkdecrypt_*.go  │
│  (C源码)              (编译工具)        (Go结构体)                │
└─────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────┐
│                   运行时加载                                      │
│                                                                 │
│  LoadKmeshTcMarkDecrypt() ──→ bpf(BPF_PROG_LOAD) ──→ 内核       │
│  (Go加载函数)                  (系统调用)              (eBPF程序) │
│                                    ↓                           │
│                              返回 prog_fd                      │
│                          (程序文件描述符)                        │
└─────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────┐
│                   使用阶段                                        │
│                                                                 │
│  decryptProg.FD() ──→ prog_fd ──→ BpfFilter.Fd ──→ 内核 TC 框架  │
│  (获取FD)             (返回FD)     (附加到过滤器)    (执行程序)   │
└─────────────────────────────────────────────────────────────────┘
```

最后的BpfFilter结构体的创建代码如下：
``` go
filter := &netlink.BpfFilter{
    FilterAttrs: netlink.FilterAttrs{
        LinkIndex: link.Attrs().Index,        // 网络接口索引
        Parent:    netlink.HANDLE_MIN_INGRESS, // 附加点
        Handle:    1,                         // 过滤器句柄
        Protocol:  unix.ETH_P_ALL,            // 处理的协议类型
        Priority:  1,                         // 优先级
    },
    Fd:           tcFd,                       // eBPF 程序文件描述符
    Name:         "tc_ingress-eth0",          // 过滤器名称
    DirectAction: true,                       // 直接动作模式
}
```



数据结构如下
``` go
type IPSecController struct {
    informer      cache.SharedIndexInformer    // K8s 资源监听器
    lister        kmeshnodeinfov1alpha1.KmeshNodeInfoLister  // 节点信息查询器
    queue         workqueue.TypedRateLimitingInterface[any]  // 工作队列
    knclient      v1alpha1_clientset.KmeshNodeInfoInterface  // K8s 客户端
    kmeshNodeInfo v1alpha1.KmeshNodeInfo       // 本地节点信息
    ipsecHandler  *IpSecHandler                // IPSec 处理器
    kniMap        *ebpf.Map                    // eBPF 映射表
    tcDecryptProg *ebpf.Program                // TC 解密程序
}

ipsecController.kmeshNodeInfo = v1alpha1.KmeshNodeInfo{
    ObjectMeta: metav1.ObjectMeta{
        Name: localNodeName,
    },
    Spec: v1alpha1.KmeshNodeInfoSpec{
        SPI:       ipsecController.ipsecHandler.Spi,  // 从加载的密钥获取 SPI
        Addresses: []string{},                        // 节点 IP 地址列表
        BootID:    localNode.Status.NodeInfo.BootID, // 节点启动 ID
        PodCIDRs:  localNode.Spec.PodCIDRs,          // Pod CIDR 范围
    },
}

```


上述内容完成的是控制平面有关于解密的逻辑，还缺少IPsec加密的逻辑。

**IPsec加密的逻辑**
首先需要了解Kubernetes中CNI插件的概念。

使用加密的设置在 enableTcMarkEncrypt 函数中进行 *kmesh/pkg/cni/plugin/plugin.go*。
- enableTcMarkEncrypt 函数中首先会获取进行标记的bpf程序，然后再获取pod虚拟网卡在主机侧的对端索引。
- 然后通过与加密同样的 ManageTCProgram 函数来附加TC加密程序。也就是说，加密程序是附加在pod虚拟网卡的出口网卡上。


#### 数据平面实现

#### 用户接口

## 跨节点 Pod 通信的内核级数据包处理流程（Claude Sonnet总结）

### 源节点数据包发送流程

#### 1. 应用程序发起连接
```
应用进程 -> connect() 系统调用 -> 内核网络栈
```

#### 2. Cgroup eBPF 拦截和处理 (CONNECT 钩子)
```c
// 在 connect() 系统调用进入内核网络栈之前被触发
cgroup_connect4_prog() {
    // 解析目标地址，构建 frontend_key
    frontend_k.addr.ip4 = orig_dst_addr.ip4;
    
    // 查找 frontend_map -> service_map -> endpoint_map -> backend_map
    frontend_manager(kmesh_ctx, frontend_v);
    
    // 重写目标地址
    SET_CTX_ADDRESS4(ctx, &kmesh_ctx.dnat_ip, kmesh_ctx.dnat_port);
}
```

#### 3. 进入内核网络栈处理
经过 eBPF 处理后，修改后的连接请求进入标准的内核网络栈：

```
Socket 层 -> TCP 层 -> IP 层 -> 路由决策
```

#### 4. 路由决策和转发
```
内核路由表查找 -> 确定出口网卡 -> 准备发送数据包
```

#### 5. TC eBPF 程序处理 (如果启用 IPsec)
在数据包即将离开网卡前：
```c
// TC egress 钩子
tc_mark_encrypt() {
    // 为跨节点流量打上加密标记
    mark = 0xe0; // egress 标记
    // 标记数据包需要 IPsec 加密
}
```

#### 6. XFRM 框架处理 (IPsec 加密)
如果启用了 IPsec：
```
数据包 -> XFRM Policy 匹配 -> XFRM State 查找 -> ESP 封装 -> 加密
```

#### 7. 物理网卡发送
```
网卡驱动 -> 物理介质 -> 网络传输
```

### 网络传输阶段

#### 8. 跨节点网络路由
```
源节点网卡 -> 交换机/路由器 -> 目标节点网卡
```

### 目标节点数据包接收流程

#### 9. 物理网卡接收
```
网络介质 -> 网卡驱动 -> 内核网络栈
```

#### 10. XFRM 框架处理 (IPsec 解密)
如果是加密流量：
```
加密数据包 -> XFRM Policy 匹配 -> ESP 解封装 -> 解密 -> 原始数据包
```

#### 11. TC eBPF 程序处理 (如果启用 IPsec)
在数据包进入网卡后：
```c
// TC ingress 钩子
tc_mark_decrypt() {
    // 处理加密标记
    mark = 0xd0; // ingress 标记
    // 标记已完成解密处理
}
```

#### 12. IP 层路由决策
```
IP 层 -> 路由表查找 -> 确定目标为本地 Pod
```

#### 13. XDP eBPF 程序处理 (授权检查)
在 Pod 网络接口上：
```c
// XDP 钩子，最早的包处理点
xdp_authz() {
    // 根据源/目标信息查找授权策略
    // 允许/拒绝流量
    return XDP_PASS; // 或 XDP_DROP
}
```

#### 14. 到达目标 Pod
```
本地路由 -> Pod 网络命名空间 -> 目标进程
```

### Service 规则与路由决策的详细影响

#### Service 负载均衡策略对路由的影响：
Kmesh 在 `service_manager()` 中实现了多种负载均衡策略，直接影响跨节点通信的路由选择：

```c
// 从 service.h 中的核心路由逻辑
switch (service_v->lb_policy) {
    case LB_POLICY_RANDOM:      // 随机选择 endpoint
    case LB_POLICY_STRICT:      // 严格本地性优先 (同节点优先)
    case LB_POLICY_FAILOVER:    // 本地性故障转移
}
```

**优先级队列机制 (PRIO_COUNT)**：
- **prio_endpoint_count[0]**: 最高优先级 (通常是同节点 endpoints)
- **prio_endpoint_count[1-6]**: 按地理位置递减的优先级
- 在 `lb_locality_failover_handle()` 中依次尝试不同优先级

#### 内核路由决策的具体过程：
1. **Frontend Map 查找**: 根据目标 IP 找到对应的 service_id
2. **Service Map 查找**: 获取负载均衡策略和 endpoint 分布
3. **Endpoint Map 查找**: 基于优先级和随机算法选择具体 backend
4. **Backend Map 查找**: 获取最终的目标 IP 和端口
5. **地址重写**: 修改 socket 的目标地址

#### 同节点连接的内核级优化：
虽然 Kmesh 中所有连接都经过 eBPF 处理，但"同节点直接连接"体现在：
- **优先级 0**: 同节点的 endpoints 被赋予最高优先级
- **路由表优化**: 内核路由表中同节点路由跳数最少
- **网络栈简化**: 无需跨网卡转发，直接通过 loopback 或 veth pair

### 关键内核处理点总结

#### eBPF 挂载点处理顺序：
1. **Cgroup Connect (源节点)**：地址重写和服务发现
2. **TC Egress (源节点)**：加密标记和 IPsec 处理  
3. **TC Ingress (目标节点)**：解密处理
4. **XDP (目标节点)**：授权检查和流量控制

#### 网络栈处理层次：
```
应用层 (connect())
    ↓
传输层 (TCP/UDP) 
    ↓  
网络层 (IP路由)
    ↓
数据链路层 (网卡驱动)
    ↓
物理层 (网络介质)
```

#### 数据包在内核中的详细处理路径：

**源节点处理路径**：
```
connect() 系统调用
    ↓
Cgroup eBPF (BPF_CGROUP_INET4_CONNECT)
    ↓ [地址重写完成]
内核 TCP/IP 协议栈
    ↓
路由子系统 (确定出口网卡)
    ↓
TC eBPF (tc_mark_encrypt) [如果启用 IPsec]
    ↓ [打上 0xe0 标记]
XFRM 子系统 [IPsec 加密]
    ↓ [ESP 封装]
网卡驱动 (物理发送)
```

**目标节点处理路径**：
```
网卡驱动 (接收数据包)
    ↓
XFRM 子系统 [IPsec 解密]
    ↓ [ESP 解封装]
TC eBPF (tc_mark_decrypt) [如果启用 IPsec]
    ↓ [处理 0xd0 标记]
路由子系统 (确定本地路由)
    ↓
XDP eBPF (xdp_authz) [授权检查]
    ↓ [XDP_PASS 或 XDP_DROP]
内核网络栈 (IP/TCP 处理)
    ↓
目标进程 socket
```

#### 内核 Netfilter 集成点：
- **PREROUTING**: XFRM 解密后的数据包路由前处理
- **FORWARD**: 跨节点转发时的策略检查
- **INPUT**: 到达本机的数据包最终处理
- **OUTPUT**: 本机发出数据包的初始处理
- **POSTROUTING**: 数据包离开本机前的最后处理

#### Kmesh 与标准 Linux 网络栈的集成：
- **不替换**标准网络栈，而是在关键点**增强**处理能力
- **eBPF 程序**在内核空间高效处理，避免用户态代理开销
- **保持兼容性**，应用程序无需修改
- **性能优化**：相比传统 sidecar 减少了用户态/内核态切换开销

这个过程展现了 Kmesh 如何通过 eBPF 在内核的多个关键点进行流量治理，实现高性能的服务网格数据平面，同时保持与标准 Linux 网络栈的完美集成。
