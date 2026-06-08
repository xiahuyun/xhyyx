+++
title = 'Kubernetes: client-go 源码剖析（二）'
date = 2023-12-16T14:25:38+08:00
categories = ["kubernetes"]
tags = ["kubernetes", "client-go"]
author = 'xhy'
+++

上接 [Kubernetes: client-go 源码剖析（一）](https://www.cnblogs.com/xingzheanan/p/17904625.html)

## 运行 `informer`

运行 `informer` 将 `Reflector`，`informer` 和 `indexer` 组件关联以实现 `informer` 流程图的流程。

### Reflector List&Watch

运行 `informer`：
```go
informer.Run(stopCh)

// client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    func() {
		...
        // 创建 DeltaFIFO 队列
		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})

		cfg := &Config{
			Queue:             fifo,
			ListerWatcher:     s.listerWatcher,
			ObjectType:        s.objectType,
			ObjectDescription: s.objectDescription,
			FullResyncPeriod:  s.resyncCheckPeriod,
			RetryOnError:      false,
			ShouldResync:      s.processor.shouldResync,
			Process:           s.HandleDeltas,
			WatchErrorHandler: s.watchErrorHandler,
		}

        // 根据 Config 创建 informer 的 controller
		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

    ...
    // goroutine 运行 processor
	wg.StartWithChannel(processorStopCh, s.processor.run)

	...
    // 运行 controller
	s.controller.Run(stopCh)
}
```

首先，创建队列 `Delta FIFO`：
```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	...
	func() {
		fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
			KnownObjects:          s.indexer,
			EmitDeltaTypeReplaced: true,
			Transformer:           s.transform,
		})
	}()
}

func NewDeltaFIFOWithOptions(opts DeltaFIFOOptions) *DeltaFIFO {
	...
	f := &DeltaFIFO{
		items:        map[string]Deltas{},
		queue:        []string{},
		keyFunc:      opts.KeyFunction,
		knownObjects: opts.KnownObjects,

		emitDeltaTypeReplaced: opts.EmitDeltaTypeReplaced,
		transformer:           opts.Transformer,
	}
	f.cond.L = &f.lock
	return f
}
```

该队列中存储的是 `Delta` 资源对象，其存储结构为：  

![client-go Delta FIFO](/images/Delta-FIFO.png)

为什么队列要设计成这个样子？因为 `informer` 在读取队列时，根据 `items` 的 action type 调用对应 `EventHandler` 的回调函数。

接下来，实例化 `informer` 的 `controller` 对象，并且调用 `controller.Run` 运行 `controller`：
```go
func (c *controller) Run(stopCh <-chan struct{}) {
	...
	r := NewReflectorWithOptions(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		ReflectorOptions{
			ResyncPeriod:    c.config.FullResyncPeriod,
			TypeDescription: c.config.ObjectDescription,
			Clock:           c.clock,
		},
	)

	...
	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```

在 `Run` 方法中创建 `Reflector` 核心组件：
```go
func NewReflectorWithOptions(lw ListerWatcher, expectedType interface{}, store Store, options ReflectorOptions) *Reflector {
	...
	// Reflector 中包括 ListerWatcher 对象和 DeltaFIFO 队列
	r := &Reflector{
		name:            options.Name,
		resyncPeriod:    options.ResyncPeriod,
		typeDescription: options.TypeDescription,
		listerWatcher:   lw,
		store:           store,
		...
	}

	...
	return r
}
```

继续进入 `wg.StartWithChannel` 中运行 `Reflector.Run`：
```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	wait.BackoffUntil(func() {
		// 调用 Reflector 的 ListAndWatch 方法
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
}

func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	...
	if fallbackToList {
		err = r.list(stopCh)
		if err != nil {
			return err
		}
	}

	...
	return r.watch(w, stopCh, resyncerrc)
}
```

在这里我们看到 `Reflector` 的 `ListAndWatch` 实现了资源的 list 和 watch 操作。这相当于 `informer` 流程图的第一步。

具体看 `Reflector` 的 `list` 方法做了什么：
```go
func (r *Reflector) list(stopCh <-chan struct{}) error {
	...
	go func() {
		...
		pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
			return r.listerWatcher.List(opts)
		}))

		list, paginatedResult, err = pager.ListWithAlloc(context.Background(), options)
	}()

	select {
	case <-stopCh:
		return nil
	case r := <-panicCh:
		panic(r)
	// 阻塞 list
	case <-listCh:
	}

	...
}
```

首先，在 `goroutine` 内调用 `pager.ListWithAlloc` 获得 list 的资源对象：
```go
func (p *ListPager) ListWithAlloc(ctx context.Context, options metav1.ListOptions) (runtime.Object, bool, error) {
	return p.list(ctx, options, true)
}

func (p *ListPager) list(ctx context.Context, options metav1.ListOptions, allocNew bool) (runtime.Object, bool, error) {
	...
	for {
		select {
		case <-ctx.Done():
			return nil, paginatedResult, ctx.Err()
		default:
		}

		obj, err := p.PageFn(ctx, options)
		...
	}

	...
}
```

`list` 方法内调用 `p.PageFn` 获得资源对象 `obj`，调用 `p.PageFn` 实际调用的是 `Reflector.listerWatcher` 对象的 `List` 方法：
```go
func (lw *ListWatch) List(options metav1.ListOptions) (runtime.Object, error) {
	return lw.ListFunc(options)
}

func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
		...
		}
	)
}
```

可以看到，这里将 `Reflector` 和前面的 `ListFunc` 回调函数关联上了，实际通过 `ClientSet` 客户端对象 list `kube-apiserver` 的资源。

### Reflector Add Object

`client-go informer` 流程的第一步实现了，那么第二步在哪呢？带着这个问题，我们继续看 `Reflector.list` 方法：
```go
func (r *Reflector) list(stopCh <-chan struct{}) error {
	...

	// 通过反射读取 list 的 meta field
	listMetaInterface, err := meta.ListAccessor(list)
	if err != nil {
		return fmt.Errorf("unable to understand list result %#v: %v", list, err)
	}

	// 读取 resource version
	resourceVersion = listMetaInterface.GetResourceVersion()
	items, err := meta.ExtractListWithAlloc(list)

	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("unable to sync list result: %v", err)
	}

	return nil
}
```

重点看 `Reflector.syncWith` 方法：
```go
func (r *Reflector) syncWith(items []runtime.Object, resourceVersion string) error {
	found := make([]interface{}, 0, len(items))
	for _, item := range items {
		found = append(found, item)
	}
	return r.store.Replace(found, resourceVersion)
}

func (f *DeltaFIFO) Replace(list []interface{}, _ string) error {
	...
	for _, item := range list {
		key, err := f.KeyOf(item)
		if err != nil {
			return KeyError{item, err}
		}
		keys.Insert(key)
		if err := f.queueActionLocked(action, item); err != nil {
			return fmt.Errorf("couldn't enqueue object: %v", err)
		}
	}

	...
	return nil
}
```

`Reflector.syncWith` 调用 `Reflector.DeltaFIFO` 的 `Replace` 方法将 list 的资源对象添加到 `DeltaFIFO` 队列中，实现 `informer` 流程的第二步。

watch 资源和 list 资源的流程类似，这里就不过多介绍了。

### informer Pop Object

`Reflector` 作为生产者将 list&watch 的资源添加到 `Delta FIFO` 队列，那么消费者在哪里使用 `Delta FIFO` 的资源呢？

`client-go` 在 `controller.Run` 中的 `controller.processLoop` 处理 `Delta FIFO` 的资源：
```go
wait.Until(c.processLoop, time.Second, stopCh)

// client-go/tools/cache/controller.go
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}
```

`controller.processLoop` 会轮询队列中的资源，当队列中有资源加入时 Pop 资源：
```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
    for {
		for len(f.queue) == 0 {
			...

            // 当队列无资源的时候协程阻塞
			f.cond.Wait()
		}

        id := f.queue[0]
		f.queue = f.queue[1:]
		depth := len(f.queue)
	    ...
		item, ok := f.items[id]
		...
		delete(f.items, id)
        ...
        err := process(item, isInInitialList)
        ...
        return item, err
    }
}
```

`DeltaFIFO.Pop` 会循环读取队列中的资源，当队列无资源时进入阻塞状态。如果队列中有资源，每次读取队列的首元素，删除队列中读取的首元素，然后调用回调函数 `PopProcessFunc` 处理读取的首元素：
```go
err := process(item, isInInitialList)

// client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	if deltas, ok := obj.(Deltas); ok {
		return processDeltas(s, s.indexer, deltas, isInInitialList)
	}
	return errors.New("object given as Process argument is not Deltas")
}
```

调用回调函数 `PopProcessFunc` 实际调用的是 `sharedIndexInformer.HandleDeltas` 方法，在该方法内处理从队列读取到的资源。

至此，实现了 `informer` 流程图的第三步。

### informer Add and Store Object

继续看 `sharedIndexInformer.HandleDeltas` 的函数 `processDeltas`：
```go
func processDeltas(handler ResourceEventHandler, clientState Store, deltas Deltas, isInInitialList bool,) error {
	for _, d := range deltas {
		obj := d.Object

		switch d.Type {
		case Sync, Replaced, Added, Updated:
			if old, exists, err := clientState.Get(obj); err == nil && exists {
				if err := clientState.Update(obj); err != nil {
					return err
				}
				handler.OnUpdate(old, obj)
			} else {
				if err := clientState.Add(obj); err != nil {
					return err
				}
				handler.OnAdd(obj, isInInitialList)
			}
		case Deleted:
			if err := clientState.Delete(obj); err != nil {
				return err
			}
			handler.OnDelete(obj)
		}
	}
	return nil
}
```

在 `processDeltas` 中根据不同的 Delta Type 执行不同的 case。我们以 `Added` type 为例查看处理流程。

首先，`clientState.Get` 从本地 `indexer` 存储中读取资源，并判断资源是否存在：
```go
// clientState.Get get 资源 obj
func (c *cache) Get(obj interface{}) (item interface{}, exists bool, err error) {
	...
	return c.GetByKey(key)
}

func (c *cache) GetByKey(key string) (item interface{}, exists bool, err error) {
	item, exists = c.cacheStorage.Get(key)
	return item, exists, nil
}

// 从 index 中读取资源
func (c *threadSafeMap) Get(key string) (item interface{}, exists bool) {
	c.lock.RLock()
	defer c.lock.RUnlock()
	item, exists = c.items[key]
	return item, exists
}
```

如果资源不存在，进入 `cache.Add`:
```go
func (c *cache) Add(obj interface{}) error {
	...
    // 添加资源到 indexer
	c.cacheStorage.Add(key, obj)
	return nil
}

func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.Update(key, obj)
}

func (c *threadSafeMap) Update(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()
	oldObject := c.items[key]
	c.items[key] = obj
	c.index.updateIndices(oldObject, obj, key)
}
```

在 `clientState.Add` 中将队列 `Delta FIFO` 读取的资源存入 `indexer` 中。

至此，完成了 `informer` 流程图的第四步和第五步。

### Event Handler

将资源存入 `indexer` 后继续往下看 `sharedIndexInformer.OnAdd` 是怎么处理 Pop 出的资源的：
```go
func (s *sharedIndexInformer) OnAdd(obj interface{}, isInInitialList bool) {
	...
	s.processor.distribute(addNotification{newObj: obj, isInInitialList: isInInitialList}, false)
}

func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for listener, isSyncing := range p.listeners {
		switch {
		case !sync:
			// non-sync messages are delivered to every listener
			listener.add(obj)
		case isSyncing:
			// sync messages are delivered to every syncing listener
			listener.add(obj)
		default:
			// skipping a sync obj for a non-syncing listener
		}
	}
}

func (p *processorListener) add(notification interface{}) {
	if a, ok := notification.(addNotification); ok && a.isInInitialList {
		p.syncTracker.Start()
	}
	p.addCh <- notification
}
```

可以看到，Pop 的资源被加入 `processorListener.addCh` 通道。

那么，通道的另一端是哪里在处理呢？  

答案在 `sharedIndexInformer.Run` 中的 `sharedProcessor.run`:
```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    ...
    wg.StartWithChannel(processorStopCh, s.processor.run)
    ...
}

func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		for listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	<-stopCh
	...
}
```

在 `sharedProcessor.run` 方法中开启两个协程分别执行 `listener.run` 和 `listener.pop` 方法。

我们先看 `listener.run` 方法：
```go
func (p *processorListener) run() {
	stopCh := make(chan struct{})
	wait.Until(func() {
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
				p.handler.OnUpdate(notification.oldObj, notification.newObj)
			case addNotification:
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				if notification.isInInitialList {
					p.syncTracker.Finished()
				}
			case deleteNotification:
				p.handler.OnDelete(notification.oldObj)
			default:
				utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
			}
		}
		close(stopCh)
	}, 1*time.Second, stopCh)
}
```

`listener.run` 从 `processorListener.nextCh` 通道中读取资源对象，根据资源对象的类型决定执行哪个 case。

前面通道的一端将 `addNotification` 加入到 `processorListener` 的 `addCh` 通道 `p.addCh <- notification`。  
这里 `processorListener` 根据 `nextCh` 通道的资源执行相应的 case。

那么 `addCh` 和 `nextCh` 的关联在哪里呢？  

我们看 `processorListener.pop`：
```go
func (p *processorListener) pop() {
	defer utilruntime.HandleCrash()
	defer close(p.nextCh) // Tell .run() to stop

	var nextCh chan<- interface{}
	var notification interface{}
	for {
		select {
		case nextCh <- notification:
			// Notification dispatched
			var ok bool
			notification, ok = p.pendingNotifications.ReadOne()
			if !ok { // Nothing to pop
				nextCh = nil // Disable this select case
			}
		case notificationToAdd, ok := <-p.addCh:
			if !ok {
				return
			}
			if notification == nil { // No notification to pop (and pendingNotifications is empty)
				// Optimize the case - skip adding to pendingNotifications
				notification = notificationToAdd
				nextCh = p.nextCh
			} else { // There is already a notification waiting to be dispatched
				p.pendingNotifications.WriteOne(notificationToAdd)
			}
		}
	}
}
```

`processorListener.pop` 的逻辑比较复杂，这里不过多介绍。重点在通过 `nextCh = p.nextCh` 将 `processorListener.nextCh` 和函数内通道 `nextCh` 关联，从而实现 `processorListener.addCh` 通道到 `processorListener.nextCh` 通道的数据传递。

了解了通道间的数据传递。我们以 `ResourceEventHandlerFuncs.OnAdd` 为例看 `client-go` 是怎么调用 `EventHandler` 的：
```go
func (p *processorListener) run() {
	...
	wait.Until(func() {
		for next := range p.nextCh {
			switch notification := next.(type) {
			case updateNotification:
				...
			case addNotification:
				p.handler.OnAdd(notification.newObj, notification.isInInitialList)
				...
			}
			...
		}
	})
}

func (r ResourceEventHandlerFuncs) OnAdd(obj interface{}, isInInitialList bool) {
	if r.AddFunc != nil {
		r.AddFunc(obj)
	}
}

func main() {
	...
	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			mObj := obj.(v1.Object)
			log.Printf("New Pod Added to Store: %s", mObj.GetName())
		},
		...
	})
}
```

可以看到，最终通过回调函数执行我们定义的 `AddFunc` handler。

至此，实现了 `informer` 流程图的第六步。

## 总结

我们通过两篇文章从源码角度介绍了 `client-go` 的流程。下面要开始 `kube-schduler` 的学习了，敬请期待...
