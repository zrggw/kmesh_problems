# kmesh 开发

mock 测试的概念

gomonkey：A Go Mocking Framework
用于模块之间解耦，然后进行单元测试

## k8s 官方库

### fake

在测试的过程中会见到这个库，所以需要进一步了解一下这个库的作用是什么。
Kubernetes提供的 mock implement of Kubernetes client interface。

**什么是client-go**


**TypedRateLimitingQueue**

A TypedRateLimitingQueue is an enhanced version of Kubernetes work queues that:

Provides type safety through Go generics (hence "Typed")
Implements rate limiting to control how fast items are processed
Prevents duplicate items in the queue
Handles retries with exponential backoff

需要阅读源码，理解其工作原理。实际上，ratelimiter会设置一个basedelay和maxdelay，并且记录失败次数，根据失败次数调整延迟时间，比如根据失败次数增加 2^failures ns。

- Get: 获取一个item

- Done: 标记一个item处理完成，出队列。我的理解是在 Get 到 Done 之间如果这个key重新被尝试加入了队列，那么会被标记为脏数据，重新加入队列

- Forget: 标记一个item不再需要处理，
  这个方法会清除该item的所有状态信息，包括失败次数。

```go
type TypedRateLimiter[T comparable] interface {
	// When gets an item and gets to decide how long that item should wait
	When(item T) time.Duration
	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for failing
	// or for success, we'll stop tracking it
	Forget(item T)
	// NumRequeues returns back how many failures the item has had
	NumRequeues(item T) int
}
```

**NewSimpleClientset**

1. client：创建一个kubernetes客户端

## 测试

测试的方法和代码可以参考kmesh中已有的代码，也可以参考 Kubernetes 项目中的代码。

**gomonkey**
gomonkey 在进行单元测试的时候需要经常被用到，所以需要了解gomonkey的使用。

gomonkey项目地址在 https://github.com/agiledragon/gomonkey

使用的时候需要注意: 

1. 确保在测试结束后调用Reset方法，以便清理所有的补丁。
2. 使用ApplyFunc或ApplyMethod来替换函数或方法的实现。
3. 可以使用ApplyFuncReturn来指定函数的返回值。
4. go test的时候需要加上 -gcflags="-all=-N -l" 参数，以便禁用编译器优化，确保gomonkey能够正常工作。

gomonkey的实现原理，见 https://bou.ke/blog/monkey-patching-in-go/

## ipsec

已发现的问题
1. 现在的ipsec配置会把node上所有的的pod都纳入xfrm policy规则中，而不是只有被纳管的namespace中的pod。
这会导致，如istiod发送回的数据包由于没有挂载加密程序所以没有被加密，而数据包来到local nic之后，会匹配到的xfrm policy中的规则，但是由于没有被加密，内核选择丢弃了这些数据。因此会出现超时错误。
可能的解决方法是，调整xfrm policy的范围，使其只包含被kmesh纳管的pod的ip地址。

2. 在node上cat /proc/net/xfrm_stat，发现很多的XfrmInTmplMismatch

3. 不同node上的pod ip地址会有重叠吗？

### cilium

#### cilium的ipsec

## E2E测试

在本地进行测试的时候，注意清空 /sys/fs/bpf 和 /mnt/kmesh_cgroup2 目录下的内容。否则可能会出现测试结果不稳定的情况，目前还不清楚为什么会出现这种情况。

```bash
umount /sys/fs/bpf
rm -rf /sys/fs/bpf/*
umount /mnt/kmesh_cgroup2
rm -rf /mnt/kmesh_cgroup2
```


**需要明确，开启ipsec之后，我们想要的是kmesh管理的所有pod接收到的数据包都是经过加密的，还是可以允许接收到的数据包可以不被加密。因为这直接关系到我们配置policy时的范围，当前我们设置的policy范围是要求所有发送往pod的数据包都经过加密。因此会把所有pod的ip地址都纳入xfrm policy规则中。**