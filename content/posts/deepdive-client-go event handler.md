+++
title = 'client-go event handler 精讲'
date = 2024-08-21T23:56:38+08:00
categories = ["client-go"]
tags = ["client-go", "kubernetes"]
summary = 'client-go event handler'
author = 'xhy'
+++

# 1. 介绍

[client-go DeltaFIFO](client-go%20DeltaFIFO%20精讲.md) 介绍了从 DeltaFIFO 中取出的资源将被放到 [indexer](client-go%20indexer%20精讲.md)
，接着交由 `processorListener` 处理。

本文继续看 `processorListener` 是如何处理 DeltaFIFO pop 的资源的。

# 2. 启动 handler

`processorListener` 的启动在 `sharedIndexInformer.Run()` 方法定义：
```aiignore
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
        ...
	// 开启协程运行 sharedIndexInformer.processor.run 方法
	wg.StartWithChannel(processorStopCh, s.processor.run)
        ...
}

func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for listener := range p.listeners {
		        // 开启协程运行 processorListener.run
			p.wg.Start(listener.run)
			
			// 开启协程运行 processorListener.pop
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh
	...
}
```

`sharedProcessor.run` 的重点在 `processorListener.pop` 和 `processorListener.run`，分别介绍如下。

## 2.1 processorListener.pop

```aiignore
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			var ok bool
			// 读取 processorListener.pendingNotifications 
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
			        // 表示 processorListener.pendingNotifications 没有资源
				nextCh = nil // Disable this select case
			}
		// DeltaFIFO 将资源信息存入 processorListener.addCh
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
			        // 如果 notification 为 nil，表示目前没有资源处理
			        // 将 processorListener.addCh 的资源赋给 notification，下一次处理该资源
				notification = notificationToAdd
				// 绑定 nextCh 和 processorListener.nextCh
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
			        // 如果 notification 不为空，表示当前有资源在处理
			        // 将 notificationToAdd 写入 processorListener.pendingNotifications
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```

这里的 `processorListener.pop()` 是两个逻辑耦合在一起的。一个是从通道 addCh 中将资源转入 nextCh 通道。另一个是如果 nextCh
的资源未能被及时处理，则将资源写入 `processorListener.pendingNotifications`，以等待 nextCh 空闲时处理。

为什么这样处理是 nextCh 和 addCh 的处理速度不一致。nextCh 的处理要慢于 addCh，所以需要将资源写入 pendingNotifications 做为缓冲。

`processorListener.pendingNotifications` 是一个 ringBuffer 循环队列，它不是并发安全的，不同于普通循环队列，它的长度是可扩容的。这是相比于通道的好处，通道的长度是固定的（但是通道是并发安全的）。

## 2.2 processorListener.run

继续看 `processorListener.run` 是如何处理资源的。

```aiignore
func (p *processorListener) run() {
	stopCh := make(chan struct{})
	wait.Until(func() {
	        // 从 processorListener.nextCh 通道中获取资源
		for next := range p.nextCh {
		        // 获取资源类型
			switch notification := next.(type) {
			case updateNotification:
			        // 处理 updateNotification 类型资源
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
			        // 处理 addNotification 类型资源
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				if notification.isInInitialList {
					p.syncTracker.Finished()
				}
			case deleteNotification:
			        // 处理 deleteNotification 类型资源
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		// the only way to get here is if the p.nextCh is empty and closed
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```

`processorListener.run` 根据不同资源类型调用 `processorListener.handler` 做相应的处理。

这里以 `processorListener.handler.OnAdd()` 方法为例看 `processorListener.handler` 对资源做了什么。
```aiignore
// processorListener.handler.OnAdd() 实际调用的是 ResourceEventHandlerFuncs.OnAdd() 方法
func (r ResourceEventHandlerFuncs) OnAdd(obj interface{}, isInInitialList bool) {
	if r.AddFunc != nil {
		r.AddFunc(obj)
	}
}
```

`ResourceEventHandlerFuncs.OnAdd()` 实际执行的是 `ResourceEventHandlerFuncs.AddFunc()` 该方法是我们在创建 informer 时注入的：
```aiignore
func main() {
	// 解析 kubeconfig
	config, err := clientcmd.BuildConfigFromFlags("", "/Users/hxia/.kube/config")
	if err != nil {
		panic(err)
	}

	// 创建 ClientSet 客户端对象
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	stopCh := make(chan struct{})
	defer close(stopCh)

	// 创建 sharedInformers
	sharedInformers := informers.NewSharedInformerFactory(clientset, time.Minute)
	// 创建 informer
	informer := sharedInformers.Core().V1().Pods().Informer()

	// 创建 Event 回调 handler
	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
	    // 注入 AddFunc 到 informer
	    // ResourceEventHandlerFuncs.AddFunc() 调用的 AddFunc 实际执行的是这里
		AddFunc: func(obj interface{}) {
			mObj := obj.(v1.Object)
			log.Printf("New Pod Added to Store: %s", mObj.GetName())
		},
	})

	// 运行 informer
	informer.Run(stopCh)
}
```

到此我们可以看到 nextCh 的资源将根据资源类型执行不同的 event handler 事件。通过该事件将资源交由业务逻辑处理。

# 3. 小结

通过两个通道实现资源的传递，这一结构有点类似于 Reflector 和 controller。  

Reflector 负责 list&watch APIServer 的资源，并将资源放入 DeltaFIFO 中，controller 负责将 DeltaFIFO 中的资源拿出来存入 indexer 和
processorListener.addCh。

类似的，processorListener.pop 负责将 processorListener.addCh 中的资源写入到 processorListener.nextCh，如果processorListener.nextCh
来不及处理，则将资源写入到 processorListener.pendingNotifications 队列。processorListener.run 负责将 processorListener.nextCh 中的
资源交由不同的 event handler 处理。

通过 processorListener.addCh 和 processorListener.nextCh 将资源的处理过程解耦，使得协程的职责更清晰。

我们可以看到模块之间可以通过通道/队列解耦，实现不同模块各司其职，较少耦合性。

除了用通道/队列接偶，模块之间解耦还有什么方式吗？比如：

- 定义接口和实现，模块之间依赖接口而不是实现。
- 通过消息总线/队列，模块之间只需要接/发消息到消息总线/队列，而不用关心对端的逻辑。
- 通过依赖注入，只需实现自己的逻辑，依赖可以通过注入解决，可用于业务逻辑和控制逻辑解耦。
- 通过代理模式或中间层解耦两个模块，模块只需要和代理或中间层通信，不需要管对方的处理逻辑，减少了耦合。
- 通过分层架构，在架构层面解耦。不同层负责处理不同事情，减少依赖。
