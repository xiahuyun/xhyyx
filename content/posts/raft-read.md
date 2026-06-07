+++
title = 'raft 读请求源码走读'
date = 2025-11-06T15:37:38+08:00
categories = ["raft"]
tags = ["raft"]
author = 'xhy'
+++

# 概述

[raft-example](https://github.com/etcd-io/etcd/tree/main/contrib/raftexample) 提供了一个简化版的 KV 存储，本文围绕 raft-example 对读请求进行源码走读。

源码版本为 `etcd release-3.6`。

# raftexample 

## 程序结构

raftexample 程序结构如下所示：
``` go
➜  raftexample git:(release-3.6) ✗ tree
.
├── Procfile
├── README.md
├── doc.go
├── httpapi.go
├── kvstore.go
├── kvstore_test.go
├── listener.go
├── main.go
├── raft.go
├── raft_test.go
└── raftexample_test.go
```

etcd raft 作为 raft 库只实现 raft 算法层的内容，对于节点通信，键值存储等都不涉及，需要用户自己提供。本文只介绍 raft 算法和存储相关内容，对节点通信等不做介绍。

## 启动 raftexample

进入 main.go 查看 raftexample 的启动流程：
``` go
func main() {  
	// 初始化启动参数
    cluster := flag.String("cluster", "http://127.0.0.1:9021", "comma separated cluster peers")  
    id := flag.Int("id", 1, "node ID")  
    kvport := flag.Int("port", 9121, "key-value server port")  
    join := flag.Bool("join", false, "join an existing cluster")  
    flag.Parse()  
  
	// 初始化 proposeC 和 confChangeC 通道
    proposeC := make(chan string)  
    defer close(proposeC)  
    confChangeC := make(chan raftpb.ConfChange)  
    defer close(confChangeC)  
  
	// 初始化 kvstore，kvstore 负责存储 KV
    var kvs *kvstore  
    getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }  
    // 创建 raftNode
    commitC, errorC, snapshotterReady := newRaftNode(*id, strings.Split(*cluster, ","), *join, getSnapshot, proposeC, confChangeC)  
  
	// 创建 KVStore
    kvs = newKVStore(<-snapshotterReady, proposeC, commitC, errorC)  
  
    // 作为客户端监听读写请求 
    serveHTTPKVAPI(kvs, *kvport, confChangeC, errorC)  
}
```

启动流程中，重点在于 `proposeC`，`confChangeC` 和 `commitC` 通道。在 `newRaftNode` 函数创建 raft 应用层节点。外部（客户端层）通过 `proposeC` 和 `confChangeC` 发送消息给 raft 应用层，客户端通过 `commitC` 接收消息。

在 `newKVStore` 函数中创建 KV 存储：

``` go
type kvstore struct {  
    proposeC    chan<- string // channel for proposing updates  
    mu          sync.RWMutex  
    kvStore     map[string]string // current committed key-value pairs  
    snapshotter *snap.Snapshotter  
}
```

raftexample 使用 map 存储键值对。

`serveHTTPKVAPI` 会创建 http handler 用来处理读写等请求。

`httpKVAPI.ServeHTTP` 中有多个请求处理分支，这里仅以 `http.MethodGet` 方法为例分析读请求是如何处理的：
``` go
func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {  
    key := r.RequestURI  
    defer r.Body.Close()  
    switch r.Method {
	case http.MethodGet:  
	    if v, ok := h.store.Lookup(key); ok {  
	       w.Write([]byte(v))  
	    } else {  
	       http.Error(w, "Failed to GET", http.StatusNotFound)  
	    }
    default:  
       w.Header().Set("Allow", http.MethodPut)  
       w.Header().Add("Allow", http.MethodGet)  
       w.Header().Add("Allow", http.MethodPost)  
       w.Header().Add("Allow", http.MethodDelete)  
       http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)  
    }  
}
```

## 读请求处理流程

进入 `h.store.Lookup`查看读请求是如何处理的：

``` go
func (s *kvstore) Lookup(key string) (string, bool) {  
    s.mu.RLock()  
    defer s.mu.RUnlock()  
    v, ok := s.kvStore[key]  
    return v, ok  
}
```

非常简单，读数据直接从 `kvStore` 中取数据即可。
