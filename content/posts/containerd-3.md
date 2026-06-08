+++
title = 'containerd 源码分析：创建 container（三）'
date = 2024-06-04T20:03:38+08:00
categories = ["kubernetes"]
tags = ["kubernetes", "containerd"]
author = 'xhy'
+++

文接 [containerd 源码分析：创建 container（二）](https://www.cnblogs.com/xingzheanan/p/18230890)

#### 启动 task

上节介绍了创建 task，task 创建之后将返回 response 给 ctr。接着，ctr 调用 `task.Start` 启动容器。 
```go
// containerd/client/task.go
func (t *task) Start(ctx context.Context) error {
	r, err := t.client.TaskService().Start(ctx, &tasks.StartRequest{
		ContainerID: t.id,
	})
	if err != nil {
		...
	}
	t.pid = r.Pid
	return nil
}

// containerd/api/services/tasks/v1/tasks_grpc.pb.go
func (c *tasksClient) Start(ctx context.Context, in *StartRequest, opts ...grpc.CallOption) (*StartResponse, error) {
	out := new(StartResponse)
	err := c.cc.Invoke(ctx, "/containerd.services.tasks.v1.Tasks/Start", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

`ctr` 调用 `contaienrd` 的 `/containerd.services.tasks.v1.Tasks/Start` 接口创建 task。进入 `containerd` 查看提供该服务的插件：
```go
// containerd/plugins/services/tasks/service.go
func (s *service) Start(ctx context.Context, r *api.StartRequest) (*api.StartResponse, error) {
	return s.local.Start(ctx, r)
}

// containerd/plugins/services/tasks/local.go
func (l *local) Start(ctx context.Context, r *api.StartRequest, _ ...grpc.CallOption) (*api.StartResponse, error) {
	t, err := l.getTask(ctx, r.ContainerID)
	if err != nil {
		return nil, err
	}
	p := runtime.Process(t)
	if r.ExecID != "" {
		if p, err = t.Process(ctx, r.ExecID); err != nil {
			return nil, errdefs.ToGRPC(err)
		}
	}
	// 启动 task: shimTask.Start
	if err := p.Start(ctx); err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	state, err := p.State(ctx)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	return &api.StartResponse{
		Pid: state.Pid,
	}, nil
}

// containerd/core/runtime/v2/shim.go
func (s *shimTask) Start(ctx context.Context) error {
	_, err := s.task.Start(ctx, &task.StartRequest{
		ID: s.ID(),
	})
	if err != nil {
		return errdefs.FromGRPC(err)
	}
	return nil
}

// containerd/api/runtime/task/v2/shim_ttrpc.pb.go
func (c *taskClient) Start(ctx context.Context, req *StartRequest) (*StartResponse, error) {
	var resp StartResponse
	if err := c.client.Call(ctx, "containerd.task.v2.Task", "Start", req, &resp); err != nil {
		return nil, err
	}
	return &resp, nil
}
```

经过 `containerd` 各个插件的层层调用，最终走到 `containerd.task.v2.Task.Start` ttrpc 服务。提供 `containerd.task.v2.Task.Start` 服务的是 `containerd-shim-runc-v2`：
```go
// containerd/cmd/containerd-shim-runc-v2/task/service.go
// Start a process
func (s *service) Start(ctx context.Context, r *taskAPI.StartRequest) (*taskAPI.StartResponse, error) {
	// 根据 task 的 StartRequest 获得 container 的 metadata
	container, err := s.getContainer(r.ID)
	if err != nil {
		return nil, err
	}

	...
	p, err := container.Start(ctx, r)
	if err != nil {
		handleStarted(container, p)
		return nil, errdefs.ToGRPC(err)
	}
	...
}
```

调用 `Container.Start` 启动容器进程：
```go
// containerd/cmd/containerd-shim-runc-v2/runc/container.go
// Start a container process
func (c *Container) Start(ctx context.Context, r *task.StartRequest) (process.Process, error) {
	p, err := c.Process(r.ExecID)
	if err != nil {
		return nil, err
	}
	if err := p.Start(ctx); err != nil {
		return p, err
	}
	...
}
```

`Container.Start` 调用 `Process.Start` 启动容器进程。启动容器后 `runc init` 将退出，将容器的主进程交由 `runc init` 的父进程 shim：
```bash
# ps -ef | grep 138915
root      138915       1  0 15:52 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id nginx1 -address /run/containerd/containerd.sock
root      138934  138915  0 15:52 ?        00:00:00 nginx: master process nginx -g daemon off;
```

通过这样的处理，容器进程就和 containerd 没关系了，容器不再受 containerd 的影响，仅和它的 shim 有关系，被 shim 管理，这也是为什么要引入 shim 的原因。

## containerd

从上述 `containerd` 创建 `container` 的分析可以看出，`containerd` 中插件之间的调用是分层的。`contianerd` 架构如下：

![containerd 架构图](/images/containerd-cncf.png)

`containerd` 创建 `container` 的示意图如下：

![containerd 示意图](/images/containerd.png)

ctr 创建的 container 的交互流程图如下：

![containerd 交互图](/images/containerd-procedure.png)

# 小结

`containerd` 源码分析系列文章介绍了 `contianerd` 是如何创建 `container` 的，完整了从 `kubernetes` 到容器创建这一条线。
