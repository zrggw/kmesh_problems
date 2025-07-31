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



