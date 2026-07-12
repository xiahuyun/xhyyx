+++
title = '深入理解 client-go 与 controller-runtime：从 informer 到调谐的性能之道'
date = 2026-07-12T17:53:38+08:00
categories = ["kubernetes", "client-go", "controller-runtime", "informer"]
tags = ["kubernetes", "client-go", "controller-runtime", "informer"]
author = 'xhy'
+++

# 前言

开发 Kubernetes CRD 资源时，绕不开 client-go 的 informer 机制。使用 client-go 本身并不困难——根据 kubebuilder 官网生成调谐框架，然后在 `Reconcile`方法中添加业务逻辑即可。

然而，只停留在“能用”的层面，很难理解其中的玄妙，出了问题心里没底。有时候代码正常工作了，自己也不清楚为什么能工作（笔者遇到过好几次）。

举个例子：调用调谐器中的 `client.List`接口读取自定义资源。写这段代码时的理解是：`client.List`从 apiserver 中读取资源，然后缓存起来。从现象上看似乎也符合预期：第一次调用花了 50ms，第二次只花了 4ms 左右。

但这个理解忽略了一个关键问题：**缓存中的数据是如何保持新鲜的？** 如果第二次只是从第一次的缓存里读数据，那这个数据岂不是永远不会更新？watch 在哪里做的？如果有 watch，它又是怎么和缓存联动的？

笔者也是因为这段代码被同事挑战了。挑战并不可怕，可怕的是自己说不清原理。于是带着问题，把 client-go 和 controller-runtime 的源码翻了一遍。其实三年前笔者就写过 [client-go 的源码分析系列](https://www.cnblogs.com/xingzheanan/p/17904625.html)文章，这次翻出来再看，还是觉得有点青涩，看来当时还是一知半解啊。

本文将继续深挖 client-go 中的核心组件，重点不是组件之间是如何工作，而是为什么这么设计，这么设计的优缺点。笔者只是翻了源码和部分 issue，并没有看过多的设计文档等内容，理解相当主观，如有不足希望能及时补充并指出。

# 架构

client-go 的架构如下：

![client-go arch](/images/client-go-arch.png)

笔者自己画了一个形象生动版的如下：

![client-go arch2](/images/client-go-arch2.png)

后面的内容都将围绕着这两张图展开，本文基于版本：
```bash
k8s.io/client-go v0.32.1  
sigs.k8s.io/controller-runtime v0.20.2
```

## informer

### reflector
#### list&watch

client-go 会起一个 reflector 协程，该协程负责：
- 全量 List 资源；
- 增量 Watch 资源；
- 指数退避保证 http 连接断了可恢复；
- 处理 List&Watch 等异常情况；
- 作为生产者将资源（事件类型+资源）存入 DeltaFIFO/RealFIFO 中；

在 [List&Watch](https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go#L470) 这里有 WatchList 和 fallbackList 两种模式，可以通过特性门控制：
- WatchList 是 client-go 客户端和 apiserver 建立一条 Watch 连接，全量资源通过该连接传给 client-go；由于 watch 和 list 共用一条连接，apiserver 在传完全量资源后会发一个 Bookmark 事件表明全量资源传完了，后面的都是 watch 资源的事件；
- fallbackList 是传统的 list 一条短连接，全量资源通过该短连接传给 client-go，后面的 watch 事件走 Watch 连接；

在 list&watch 这里有一些面试官特别喜欢问的问题：
1. 如果连接长时间断开会发生什么？
2. watch 的 resourceversion 会溢出吗？

等等...

我们先看第一个问题，如果连接长时间断开。client-go 会退回到指数退避重试，最大重试时间 1000s(默认)，直到建立连接。此时根据旧的 resourceversion watch 资源，如果 resourceversion 在 apiserver 已被清理则收到 apiserver  410 Gone 错误，client-go 则退回到全量 list 资源阶段，保证资源和 Kubernetes 集群同步。

在重连期间，缓存中缓存的是旧数据。已入 DeltaFIFO/RealFIFO 的资源事件会继续处理。

重连期间的事件不会被收到，也不会补偿——事件丢失了，但资源的状态不会丢失，因为后续的全量 List 会补齐。

比如，重联期间发生了删除了资源，该资源的删除事件并不会被 client-go 接收，后续也不会补偿，这个事件丢了。但是在重联后，全量 list 资源，发现缓存中的资源在 list 列表已不存在，client-go 会自己发一个 Delete 事件至 FIFO 用于删除缓存中的资源。

所以，重联期间会丢事件，但是资源的状态并不会丢失，还是会保持一致性的。

这种一致性也是最终一致性，什么时候是最终呢？可能是时间的尽头吧。

第二个问题一定要用数据说话。笔者曾被面试官问过这个问题，当时回答得并不理想。

Kubernetes 的 resourceversion 对应的是 etcd 中的 mod revision。它是集群集单调递增整数，类型为 int64。

int64 约等于 9.22 * 10 的 18 次方，如果一个 10w pod 的大型集群，每秒写入约 10000/s QPS，写完 int64 需要约 92 万年:

| **<br><br>集群规模<br><br>** | **<br><br>写入 QPS<br><br>** | **<br><br>跑满 int64 需要<br><br>** |
| ------------------------ | -------------------------- | ------------------------------- |
| 中型（1k Pod）               | ~100/s                     | ~2.9 × 10¹⁶ 秒 ≈ **920 万年**​     |
| 大型（10w Pod，高频控制器）        | ~10,000/s                  | ~2.9 × 10¹³ 秒 ≈ **92 万年**​      |
| etcd 官方压测极限              | ~100,000/s                 | ~2.9 × 10¹² 秒 ≈ **9.2 万年**​     |

### FIFO

这里没用 DeltaFIFO 而是用的 FIFO 是因为 client-go 新增了 RealFIFO 队列，在介绍 RealFIFO 之前先简要介绍下 DeltaFIFO 事件队列。

client-go 中的队列根据不同事件类型做不同处理，如 Delete 事件，需要删除缓存中的资源，回调 Delete 事件给 eventhandler 进行业务逻辑处理。

传统的 DeltaFIFO 将资源的名字作为 key，资源的不同事件可以合并。如资源 A： Add, Updated，Updated 可以合并为 Add 和 Updated 两个事件。

这么做的好处是可以合并同一类资源事件，减少调谐的频率。

![DeltaFIFO](/images/Delta-FIFO.png)

缺点也很明显，主要有两条：

1. 无法表示真实的资源顺序关系

如果有两个资源 A 和 B：
```bash
A:
t1: Add
t2: Update
t4: Update

B:
t3: Update
```

B 在 t3 时刻更新，但是在 queue 队列中 B 只能位于 A 之前或之后，消费者在处理时并不会先处理 A 的 Add 和 Update 事件在去处理 B 的 Update 事件，即便这是真实的事件顺序。

2. 用 key 入队列，只是全量 List 一个 Add 事件就会创建大量资源 key 入队列，对性能影响较大。

client-go 自 v1.33 后开始引入 RealFIFO 事件队列，顾名思义 RealFIFO 能保持真实的事件顺序。

它以事件类型为 Key，优点是能保证真实的事件顺序，缺点是不会做事件的合并（很难说这是不是缺点）。

这里用队列的好处我想主要有两个，一个是削峰，一个是解耦。

那么为什么不用通道而是用队列呢？通道天然也可以表示生产/消费这种模式啊。
通道过于单一，对于 DeltaFIFO，通道无法做到合并同类型事件，对于 RealFIFO，通道无法做到合并同资源，等等。


在队列的另一端，processor 负责消费队列中的事件，并根据事件类型对缓存做对应的操作。

### processor

processor 主要做了三类事：
1. 消费 FIFO 中的事件，并缓存；
2. 触发 processorListener 回调事件；
3. 管理 processorListener 生命周期；

client-go 的命名还是非常地道的。这里不叫 distributer，不叫 listener，不叫 manager，而是叫 processor， 有点意思。

FIFO 的消费者会阻塞等待 FIFO 有事件资源，如果没有则陷入阻塞休眠，不会忙等待，占用 CPU 资源。
FIFO 生产者发送完数据之后会唤醒消费者，让它起来干活。

这里也有一个问题，消费者可以多个并发消费吗？目前的 client-go 都是单消费者。
如果多消费者并发消费，一个非常重要的问题是消费顺序问题。

举例如下：
```bash
FIFO: [Delete A, Update B, Add A]

consumer1:
Delete A

consumer2:
Add A

consumer3:
Update B
```

由于并发消费，无法保证顺序性。可能会造成：
```bash
sequence1: Delete A, Add A, Update B
sequence2: Delete A, Update B， Add A

sequence3: Add A, Update B, Delete A
sequence4: Add A, Delete A, Update B

sequence5: Update B, Delete A, Add A
sequence6: Update B, Add A, Delete A
```

等六种情况，更别提可能涉及到共享资源的竞态问题了。

对于不同资源事件可能还好，对于同资源事件的顺序不一致就是错误了。我们可以对资源加 hash 映射到指定的消费者来保证顺序性，但是对于不同资源事件依然保证不了真实顺序。

这也是为什么用单消费者反而简单的原因，另一方面性能瓶颈往往在业务逻辑，这里的生产/消费多数情况是可以做到平衡的。

FIFO 中的资源事件被 processor 拿到，processor 会通知它管理的 processorListener 做相应处理。

### processorListener

我们先抛开 client-go 的设计，如果让我们自己去设计实现一个 listener 该如何设计呢？

我想我可能会用三个通道来接收不同事件（onAdd/onUpdate/onDelete）类型（或者一个通道+反射），然后根据该事件类型回调相应的事件处理函数。

这种方式可以工作，但是有问题。问题在于可能会阻塞 processor，processor 作为消息的生产者并且是 listener 的管理者，必须做到无阻塞，否则对性能影响很大。

而且 listener 离业务侧代码很近，很容易由于消费慢导致 processor 阻塞。即便用有缓冲通道也是治标不治本。

client-go 的实现非常精妙，它引入一个动态环形队列作为缓冲区，解耦 processor 和 listener。

这个动态环形队列是动态增长可复用的，这里的动态增长很重要。但是这个空间并不会动态缩小的，意味着可能会造成环形队列占用的内存越来越大。如果占用很大呢？那就不是环形队列的问题了，而是要解决为什么消费端消费不及时，为什么生产端生产过猛的问题了（一般不会是生产端的问题）。

对于有些简单的业务场景到这里就够了，可以在 eventhandler 里处理业务逻辑了。对于需要调谐资源（Operator 模式）的场景那么还需要 controller-runtime 的助力。

## controller-runtime

controller-runtime 解决的是资源调谐的问题。我们继续看它是如何做到调谐资源的。

首先，调谐资源主要负责：
1. 重复调谐，业务侧可能需要持续操作资源直到满足条件；
2. 外部限流，防止过多的请求打爆 apiserver；
3. 内部限流，对失败资源进行限流防止阻塞其它正常资源调谐等；
4. 指标监控，暴露调谐指标用于分析调谐性能等；

等等。

我们接着架构图继续往下看。

### Delaying Queue

延迟队列首先是一个队列，这一正确的废话表明资源的顺序性是首要需要保证的事情。

举例来说，调谐者并发调谐从 Delaying Queue 中获取资源，如果同一资源记在调谐者 A 出现，又在调谐者 B 出现，那么将无法保证调谐的顺序性。

由于调谐者是业务逻辑，并发是必须要的，对提升性能很有帮助。

解决这样的问题就得从源头入手，限制队列中只会有一个同名的资源在处理，解决方式是引入 dirty/processing 和 queue 结构，示意图如下：

![DelayingQueue](/images/delaying-queue.png)

[controller-runtime 框架如何保证并发性](https://juejin.cn/post/7344259131714568227) 这篇文章介绍的很好，看的很舒服，笔者不想重复造轮子，有需要进一步了接的可以看看。

注意这里解决的问题是同名资源的顺序性问题，不同资源由于并发导致调谐顺序不一致也没有影响。

队列我们说了，延迟又在哪呢？或者说为什么需要延迟呢？

主要有几点：
1. 调谐失败的资源需要进入队列重新调谐，如果不延迟调谐会造成队列拥堵，拖慢调谐正常资源；
2. 调谐失败资源频繁调谐可能会打爆 apiserver；
3. 正常资源如果不加延迟限制可能会打爆 apiserver，消耗 CPU/内存 等；

client-go 默认实现复合限流器：
```go
func NewTypedRateLimitingQueueWithConfig[T comparable](rateLimiter TypedRateLimiter[T], config TypedRateLimitingQueueConfig[T]) TypedRateLimitingInterface[T] {  
    ...
    if config.DelayingQueue == nil {  
       config.DelayingQueue = NewTypedDelayingQueueWithConfig(TypedDelayingQueueConfig[T]{  
          Name:            config.Name,  
          MetricsProvider: config.MetricsProvider,  
          Clock:           config.Clock,  
       })  
    }  
  
	// 复合限流器用于限流（延迟）
    return &rateLimitingType[T]{  
       TypedDelayingInterface: config.DelayingQueue,  
       rateLimiter:            rateLimiter,  
    }  
}
```

限流器限制资源多久才能重入队列。需要重入的资源通过 `waitingForAdd` 通道被 `waitingLoop` 消费，`waitingLoop` 根据资源的延迟时间将资源插入/更新到小堆中。接着从小堆取出延迟时间到的资源重新添加到延迟队列中处理。

这里很直觉的思考是起两个协程，一个用来插入/更新资源到堆，一个用来从堆中取出到期资源，两个协程通过锁来防止竞态。

client-go 给出了一种优雅的方案，通过单协程事件循环避免锁竞争，且心跳定时器保证了即使没有新事件，堆顶到期元素也能被及时处理。代码在 [这里](https://github.com/kubernetes/client-go/blob/master/util/workqueue/delaying_queue.go#L276)，读者可细细品读体会一下。

### informer

这里的 informer 并不是介绍 informer 的组件，而是想讲一个问题：controller-runtime 的 informer 是在哪里创建的。

这也是开篇笔者被质疑的点。

同事说自定义资源在注册时 controller-runtime 就会在背后创建 informer：
```go
func init() {  
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))  
  
	// 注册资源
    utilruntime.Must(webappv1.AddToScheme(scheme))  
    utilruntime.Must(webappv1.TestAddToScheme(scheme))  
    // +kubebuilder:scaffold:scheme  
}
```

笔者当时不知道，直到深挖了源码才知道，这里的注册只是告诉 client-go 资源的 gvk 和对象的转换关系等，解决的是资源的转换/序列化/反序列化问题。其并没有创建 informer。

实际上 informer 是基于懒加载模式创建的，有几种方式会创建 informer：

1. mgr.GetCache().GetInformer

示例如下：
```go
guestbookInformer, err := mgr.GetCache().GetInformer(context.TODO(), &webappv1.Guestbook{})  
if err != nil {  panic(err) }  

guestbookInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{  
    AddFunc: func(obj interface{}) {       
	    u := obj.(*webappv1.Guestbook)       
	    name := u.GetName()       
	    fmt.Printf("Added %s to guestbook\n", name)    },    
	UpdateFunc: func(old, new interface{}) {},    
	DeleteFunc: func(obj interface{}) {},
	}
)
```

2. mgr.GetClient().List

```go
var guestbookList webappv1.GuestbookList  
if err := mgr.GetClient().List(context.TODO(), &guestbookList, client.Limit(100)); err != nil {  
    panic(err)
}
```

笔者本意调用 client.List 方式获取资源，背后误打误撞创建了 informer 而不自知哈哈。

3. Reconciler.SetupWithManager

```go
func (r *GuestbookReconciler) SetupWithManager(mgr ctrl.Manager) error {  
    return ctrl.NewControllerManagedBy(mgr).  
       For(&webappv2.Guestbook{}).  
       Named("guestbook").  
       WithEventFilter(predicate.Funcs{  
          UpdateFunc: func(e event.UpdateEvent) bool {  
             oldOld := e.ObjectOld.(*webappv2.Guestbook)  
             newOld := e.ObjectNew.(*webappv2.Guestbook)  
             fmt.Println(oldOld.Labels)  
             fmt.Println(newOld.Labels)  
             return false  
          },  
       }).  
       Complete(r)  
}
```

这是“正统”的创建 informer 方式。

注意前两种只是创建了 informer，list&watch 资源。但是资源回调到延迟队列等等链路并没有建立，informer 的这条链路还是通过 `SetupWithManager` 建立的。

# 性能

到这里基本我们已经介绍完了 client-go 中组件的设计要点，不过在实际开发时光了解这些设计要点还不够。还需要：
- 了解 client-go 的监控指标，根据监控指标判断性能问题；
- 根据 pprof 排查 业务 CPU/内存，协程泄漏等问题；

总之，核心是要写出优雅，靠谱的业务代码。相关资料可参考 [Performance Tuning for controller-runtime: Concurrency, Client QPS, and Cache](https://www.golinuxcloud.com/controller-runtime-performance-tuning/)，写的很好，这里就不赘述了。

![performance](/images/performance.png)

# 总结

本文围绕架构图介绍了 client-go 中核心组件的设计要点，不同于源码走读系列，本文更多关注为什么这么设计，有何优缺点角度分析，希望读者能有所收获。

# 参考资料

- [controller-runtime 框架如何保证并发性](https://juejin.cn/post/7344259131714568227)
-  [kubebuilder book](https://book.kubebuilder.io/architecture.html)
- [Go官方time/rate限流器的使用详解与实战](https://www.trae.cn/article/660504066)
- [Performance Tuning for controller-runtime: Concurrency, Client QPS, and Cache](https://www.golinuxcloud.com/controller-runtime-performance-tuning/)

