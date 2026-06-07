+++
title = 'client-go DeltaFIFO 精讲'
date = 2024-08-21T23:56:38+08:00
categories = ["client-go"]
tags = ["client-go", "kubernetes"]
author = 'xhy'
+++

# 0. 介绍

[client-go list&watch 精讲](./client-go%20list%20&%20watch%20精讲.md) 介绍过 Reflector 通过 list&watch 实时获取
Kubernetes 资源的信息。并且作为生产者将资源信息存到 DeltaFIFO 中。

那么消费者是在哪里定义的呢？本文围绕 DeltaFIFO 继续介绍是哪个模块消费了 DeltaFIFO 中的资源。

# 1. controller

在 `controller.Run` 方法中
```aiignore
func (c *controller) Run(stopCh <-chan struct{}) {
	...
	// 创建 Reflector 对象
	r := NewReflectorWithOptions(
		...
	)
	...

	var wg wait.Group

        // 开启协程运行 Reflector
	wg.StartWithChannel(stopCh, r.Run)

        // 运行 controller.processLoop
	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```

`controller.processLoop` 将作为消费者处理 DeltaFIFO 中的资源：
```aiignore
func (c *controller) processLoop() {
        // 正常状态下 processLoop 是一个永不退出的函数
	for {
	        // 从 DeltaFIFO 中 Pop 资源
	        // 将资源送入 PopProcessFunc 中处理
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
		        // 如果接收到 ErrFIFOClosed：DeltaFIFO: manipulating with closed queue 则退出
			if err == ErrFIFOClosed {
				return
			}
			// 如果接收到 RetryOnError 则将资源重新加入到 DeltaFIFO 中等待下一次重新处理
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}

func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			// When Close() is called, the f.closed is set and the condition is broadcasted.
			// Which causes this loop to continue and return from the Pop().
			if f.closed {
				return nil, ErrFIFOClosed
			}

                        // 如果 DeltaFIFO 中无数据可消费，则调用 f.cond.Wait() 陷入阻塞，等待被唤醒
                        // 前面的调用 f.cond.Brondcast 就是唤醒这里的协程
			f.cond.Wait()
		}
		
		// 取出 DeltaFIFO 的首资源 key，保证资源处理的顺序性
		id := f.queue[0]
		f.queue = f.queue[1:]
		depth := len(f.queue)
		
		// 拿到资源信息
		item, ok := f.items[id]
		if !ok {
			// This should never happen
			klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
			continue
		}
		
		// 删除 DeltaFIFO 中的资源，注意这里删除的是资源，不是资源对应的 key，key 还保留在 DeltaFIFO 中
		delete(f.items, id)
		
		// 处理资源 item
		err := process(item, isInInitialList)
		// 如果资源处理出错，需要将资源重新放回 DeltaFIFO 中
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
```

调用 `process(item, isInInitialList)` 实际调用的是 `sharedIndexInformer.HandleDeltas`：

```aiignore
func (s *sharedIndexInformer) HandleDeltas(obj interface{}, isInInitialList bool) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	if deltas, ok := obj.(Deltas); ok {
	    // 获取资源的 Delta 结构，并交由 processDeltas 处理
		return processDeltas(s, s.indexer, deltas, isInInitialList)
	}
	return errors.New("object given as Process argument is not Deltas")
}

func processDeltas(...) error {
	// from oldest to newest
	// deltas 记录的是对象的顺序更新状态
	for _, d := range deltas {
		obj := d.Object

		switch d.Type {
		// 对于 Sync, Replaced, Added, Updated 状态进入一样的逻辑处理
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
		// 处理 Deleted 资源变更状态
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

`processDeltas` 主要是调用 `clientState` 和 `handler` 处理资源。分别看 `clientState` 和 `handler` 做了什么。

## 1.1 clientState

以 Add 类型为例，进入 `clientState.Add`:
```aiignore
// 调用 clientState.Add 实际调用的是 cache.Add 方法
// cache 是 client-go 的缓存实现
func (c *cache) Add(obj interface{}) error {
        // 获取对象对应的 key 信息 
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	
	// 调用 c.cacheStorage.Add 方法
	c.cacheStorage.Add(key, obj)
	return nil
}

// c.cacheStorage.Add 实际调用的是 threadSafeMap.Add 方法
func (c *threadSafeMap) Add(key string, obj interface{}) {
	c.Update(key, obj)
}

func (c *threadSafeMap) Update(key string, obj interface{}) {
	c.lock.Lock()
	defer c.lock.Unlock()
	// 获取缓存中 key 对应的资源
	oldObject := c.items[key]
	// 将新资源赋给 缓存中的 items
	c.items[key] = obj
	// 更新 indexer 中资源的信息，indexer 包括缓存 threadSafeMap
	c.index.updateIndices(oldObject, obj, key)
}
```

`clientState` 根据资源的状态类型到缓存中处理，资源存储在缓存 threadSafeMap 的 items 中，threadSafeMap 是一个并发
安全的 map，内部通过加锁实现并发(不像 Go 自带的并发安全 map，通过两个 map 实现)。

这里有个组件需要注意的是 indexer，我们先不过多介绍发散，在下一篇文章中会重点介绍 indexer 组件。

## 1.2 handler

继续查看 `handler.OnAdd` 实现：
```aiignore
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
			// 调用 listener.add 处理资源
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
	// 将包装资源的 notification 写到 processorListener.addCh 通道
	p.addCh <- notification
}
```

`handler.OnAdd` 将资源写入到 `processorListener` 的 addCh 通道。

这里不继续探究哪里接收 `processorListener.addCh` 通道的 notification，在后面的文章中会重点介绍。

# 3. 小结

本文重点介绍了 `controller` 作为消费者是如何消费 `DeltaFIFO` 中的资源的。同时也可以看出在缓存中是不带事件变更类型的，
缓存中存储的只是资源的 item 信息。

controller 根据 DeltaFIFO 中不同的资源类型做相应的处理，也侧面说明了 DeltaFIFO 为什么要 Delta 结构而不用普通队列。

从 controller 可以看出更新到缓存中的永远是最新的资源，保证了处理资源的一致性。

回头看，这里的逻辑是 [client-go 架构图](https://github.com/TroyXia/blogs/blob/main/operator/client-go/client-go%20list%20%26%20watch%20%E7%B2%BE%E8%AE%B2.md) 的第四步和第六步。

