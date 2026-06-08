+++
title = 'containerd 源码分析：创建 container（二）'
date = 2024-06-04T15:23:38+08:00
categories = ["kubernetes"]
tags = ["kubernetes", "containerd"]
author = 'xhy'
+++

文接 [containerd 源码分析：创建 container（一）](https://www.cnblogs.com/xingzheanan/p/18230301)

### 创建容器进程

创建 container 成功后，接着创建 task, task 将根据 container metadata 创建容器进程。

#### 创建 task

进入 `tasks.Newtask` 创建 task：  
```go
// containerd/cmd/ctr/commands/tasks/tasks_unix.go
func NewTask(ctx gocontext.Context, client *containerd.Client, container containerd.Container, checkpoint string, con console.Console, nullIO bool, logURI string, ioOpts []cio.Opt, opts ...containerd.NewTaskOpts) (containerd.Task, error) {
   ...
   t, err := container.NewTask(ctx, ioCreator, opts...)
	if err != nil {
		return nil, err
	}
   ...
}

// containerd/client/container.go
func (c *container) NewTask(ctx context.Context, ioCreate cio.Creator, opts ...NewTaskOpts) (_ Task, err error) {
   ...
   t := &task{
		client: c.client,
		io:     i,
		id:     c.id,
		c:      c,
	}
	...
	response, err := c.client.TaskService().Create(ctx, request)
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}
	t.pid = response.Pid
	return t, nil
}
```

类似创建 container，这里调用 `container.client.TaskService().Create` 创建 task：  
```go
// containerd/api/services/tasks/v1/tasks_grpc.pg.go
func (c *tasksClient) Create(ctx context.Context, in *CreateTaskRequest, opts ...grpc.CallOption) (*CreateTaskResponse, error) {
	out := new(CreateTaskResponse)
	err := c.cc.Invoke(ctx, "/containerd.services.tasks.v1.Tasks/Create", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

调用 `/containerd.services.tasks.v1.Tasks/Create` grpc 接口创建 task。查看 containerd 中提供该服务的插件。  
```go
// containerd/plugins/services/tasks/service.go
func init() {
	registry.Register(&plugin.Registration{
		Type: plugins.GRPCPlugin,
		ID:   "tasks",
		Requires: []plugin.Type{
			plugins.ServicePlugin,
		},
		InitFn: func(ic *plugin.InitContext) (interface{}, error) {
			// plugins.ServicePlugin: io.containerd.service.v1
			// services.TasksService: tasks-service
			i, err := ic.GetByID(plugins.ServicePlugin, services.TasksService)
			if err != nil {
				return nil, err
			}
			return &service{local: i.(api.TasksClient)}, nil
		},
	})
}
```

containerd 中提供该服务的是 `io.containerd.grpc.v1.tasks` 插件。调用插件对象 `service` 的 `Create` 方法创建 task：
```go
// containerd/plugins/services/tasks/service.go
func (s *service) Create(ctx context.Context, r *api.CreateTaskRequest) (*api.CreateTaskResponse, error) {
	return s.local.Create(ctx, r)
}
```

`service` 调用 `local` 对象的 `Create` 方法创建 task。`local` 是 `io.containerd.service.v1.tasks-service` 插件的实例化对象：
```go
// containerd/plugins/services/tasks/local.go
func init() {
	registry.Register(&plugin.Registration{
		Type:     plugins.ServicePlugin,
		ID:       services.TasksService,
		Requires: tasksServiceRequires,
		Config:   &Config{},
		InitFn:   initFunc,
	})

	timeout.Set(stateTimeout, 2*time.Second)
}

func initFunc(ic *plugin.InitContext) (interface{}, error) {
	config := ic.Config.(*Config)

	// plugins.RuntimePluginV2: io.containerd.runtime.v2
	v2r, err := ic.GetByID(plugins.RuntimePluginV2, "task")
	if err != nil {
		return nil, err
	}

	// plugins.MetadataPlugin: io.containerd.metadata.v1
	m, err := ic.GetSingle(plugins.MetadataPlugin)
	if err != nil {
		return nil, err
	}
	...
	db := m.(*metadata.DB)
	l := &local{
		containers: metadata.NewContainerStore(db),
		store:      db.ContentStore(),
		publisher:  ep.(events.Publisher),
		monitor:    monitor.(runtime.TaskMonitor),
		v2Runtime:  v2r.(runtime.PlatformRuntime),
	}
	...
}
```

进入 `local.Create`：
```go
// containerd/plugins/services/tasks/local.go
func (l *local) Create(ctx context.Context, r *api.CreateTaskRequest, _ ...grpc.CallOption) (*api.CreateTaskResponse, error) {
	// 从 boltDB 中获取 container metadata
	container, err := l.getContainer(ctx, r.ContainerID)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	...
	rtime := l.v2Runtime
	...
	c, err := rtime.Create(ctx, r.ContainerID, opts)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	...
}
```

`local.Create` 首先获取 `boltDB` 中的 container 信息， 接着调用 `local.v2Runtime.Create` 创建 task。v2Runtime 是 `io.containerd.runtime.v2.task` 插件的实例：
```go
// containerd/core/runtime/v2/manager.go
func init() {
	registry.Register(&plugin.Registration{
		Type: plugins.RuntimePluginV2,
		ID:   "task",
		...
		InitFn: func(ic *plugin.InitContext) (interface{}, error) {
			...
			// 获取 metadata 插件的实例
			m, err := ic.GetSingle(plugins.MetadataPlugin)
			if err != nil {
				return nil, err
			}
			...
			cs := metadata.NewContainerStore(m.(*metadata.DB))
			ss := metadata.NewSandboxStore(m.(*metadata.DB))
			events := ep.(*exchange.Exchange)

			shimManager, err := NewShimManager(ic.Context, &ManagerConfig{
				Root:         ic.Properties[plugins.PropertyRootDir],
				State:        ic.Properties[plugins.PropertyStateDir],
				Address:      ic.Properties[plugins.PropertyGRPCAddress],
				TTRPCAddress: ic.Properties[plugins.PropertyTTRPCAddress],
				Events:       events,
				Store:        cs,
				SchedCore:    config.SchedCore,
				SandboxStore: ss,
			})
			if err != nil {
				return nil, err
			}

			return NewTaskManager(shimManager), nil
		},
		...
	})
}

func NewShimManager(ctx context.Context, config *ManagerConfig) (*ShimManager, error) {
	...
	m := &ShimManager{
		root:                   config.Root,
		state:                  config.State,
		containerdAddress:      config.Address,
		containerdTTRPCAddress: config.TTRPCAddress,
		shims:                  runtime.NewNSMap[ShimInstance](),
		events:                 config.Events,
		containers:             config.Store,
		schedCore:              config.SchedCore,
		sandboxStore:           config.SandboxStore,
	}
	...
	return m, nil
}

func NewTaskManager(shims *ShimManager) *TaskManager {
	return &TaskManager{
		manager: shims,
	}
}
```

`io.containerd.runtime.v2.task` 插件的实例是 `TaskManger`，其中包括 shims（垫片）。调用 `local.v2Runtime.Create` 实际调用的是 `TaskManager.Create`：
```go
// containerd/core/runtime/v2/manager.go
func (m *TaskManager) Create(ctx context.Context, taskID string, opts runtime.CreateOpts) (runtime.Task, error) {
	// step1: 创建 shim
	shim, err := m.manager.Start(ctx, taskID, opts)
	if err != nil {
		return nil, fmt.Errorf("failed to start shim: %w", err)
	}

	// step2：new shim task
	shimTask, err := newShimTask(shim)
	if err != nil {
		return nil, err
	}

	// step3: 创建 task
	t, err := shimTask.Create(ctx, opts)
	if err != nil {
		...
	}

	return t, nil
}
```

首先，调用 `TaskManager.manager.Start` 启动 shim：
```go
// containerd/core/runtime/v2/manager.go
func (m *ShimManager) Start(ctx context.Context, id string, opts runtime.CreateOpts) (_ ShimInstance, retErr error) {
	// 创建 bundle，bundle 是包含 shim 信息的对象
	bundle, err := NewBundle(ctx, m.root, m.state, id, opts.Spec)
	if err != nil {
		return nil, err
	}
	...
	// 启动 shim
	shim, err := m.startShim(ctx, bundle, id, opts)
	if err != nil {
		return nil, err
	}
	...
	// 将启动的 shim 添加到 TaskManager 的 shims 中
	if err := m.shims.Add(ctx, shim); err != nil {
		return nil, fmt.Errorf("failed to add task: %w", err)
	}
	...
}
```

进入 `ShimManager.startShim` 查看启动 shim 的逻辑：
```go
// containerd/core/runtime/v2/manager.go
func (m *ShimManager) startShim(ctx context.Context, bundle *Bundle, id string, opts runtime.CreateOpts) (*shim, error) {
	...
	// 启动 shim 的 binary 对象
	b := shimBinary(bundle, shimBinaryConfig{
		runtime:      runtimePath,
		address:      m.containerdAddress,
		ttrpcAddress: m.containerdTTRPCAddress,
		schedCore:    m.schedCore,
	})
	// binary 对象启动 shim
	shim, err := b.Start(ctx, protobuf.FromAny(topts), func() {
		...
	})
	...
}

// containerd/core/runtime/v2/binary.go
func (b *binary) Start(ctx context.Context, opts *types.Any, onClose func()) (_ *shim, err error) {
	...
	cmd, err := client.Command(
		ctx,
		&client.CommandConfig{
			// runtime: /usr/bin/containerd-shim-runc-v2
			Runtime:      b.runtime,
			Address:      b.containerdAddress,
			TTRPCAddress: b.containerdTTRPCAddress,
			Path:         b.bundle.Path,
			Opts:         opts,
			Args:         args,
			SchedCore:    b.schedCore,
		})
	if err != nil {
		return nil, err
	}
	...
	out, err := cmd.CombinedOutput()
	if err != nil {
		return nil, fmt.Errorf("%s: %w", out, err)
	}
	...
}
```

shim 通过命令行执行二进制文件的方式，启动 container 对应的 runtime `/usr/bin/containerd-shim-runc-v2`。查看启动的 shim：
```bash
# ps -ef | grep nginx1
root     4144414 4144319  0 10:20 ?        00:00:00 /root/go/src/containerd/cmd/ctr/__debug_bin3399182742 run docker.io/library/nginx:alpine nginx1
root     4147233       1  0 10:49 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id nginx1 -address /run/containerd/containerd.sock
```

启动的 shim 进程为 4147233，其父进程是 root（1）进程。

shim 提供 ttrpc 服务，负责和 containerd 进行通信：

![containerd 和 shim 交互](/images/containerd-shim.png)

`TaskManager.manager.Start` 启动了 shim。接着在 `TaskManager.Create` 中调用 `newShimTask` 实例化 task。
```go
// containerd/core/runtime/v2/shim.go
func newShimTask(shim ShimInstance) (*shimTask, error) {
	taskClient, err := NewTaskClient(shim.Client(), shim.Version())
	if err != nil {
		return nil, err
	}

	return &shimTask{
		ShimInstance: shim,
		task:         taskClient,
	}, nil
}

// containerd/core/runtime/v2/bridge.go
func NewTaskClient(client interface{}, version int) (TaskServiceClient, error) {
	switch c := client.(type) {
	case *ttrpc.Client:
		switch version {
		case 2:
			return &ttrpcV2Bridge{client: v2.NewTaskClient(c)}, nil
		case 3:
			return v3.NewTTRPCTaskClient(c), nil
		default:
			return nil, fmt.Errorf("containerd client supports only v2 and v3 TTRPC task client (got %d)", version)
		}

	...
	}
}
```

task 包括和 shim 连接的 client `ttrpcV2Bridge`，它们通过 ttrpc 建立连接。

继续在 `TaskManager.Create` 中调用 `shimTask.Create` ：
```go
// containerd/core/runtime/v2/shim.go
func (s *shimTask) Create(ctx context.Context, opts runtime.CreateOpts) (runtime.Task, error) {
	...
	request := &task.CreateTaskRequest{
		ID:         s.ID(),
		Bundle:     s.Bundle(),
		Stdin:      opts.IO.Stdin,
		Stdout:     opts.IO.Stdout,
		Stderr:     opts.IO.Stderr,
		Terminal:   opts.IO.Terminal,
		Checkpoint: opts.Checkpoint,
		Options:    protobuf.FromAny(topts),
	}
	...
	_, err := s.task.Create(ctx, request)
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}

	return s, nil
}
```

进入 `shimTask.task.Create`：
```go
// containerd/core/runtime/v2/bridge.go
func (b *ttrpcV2Bridge) Create(ctx context.Context, request *api.CreateTaskRequest) (*api.CreateTaskResponse, error) {
	resp, err := b.client.Create(ctx, &v2.CreateTaskRequest{
		ID:               request.GetID(),
		Bundle:           request.GetBundle(),
		Rootfs:           request.GetRootfs(),
		Terminal:         request.GetTerminal(),
		Stdin:            request.GetStdin(),
		Stdout:           request.GetStdout(),
		Stderr:           request.GetStderr(),
		Checkpoint:       request.GetCheckpoint(),
		ParentCheckpoint: request.GetParentCheckpoint(),
		Options:          request.GetOptions(),
	})

	return &api.CreateTaskResponse{Pid: resp.GetPid()}, err
}

// containerd/api/runtime/task/v2/shim_ttrpc.pb.go
func (c *taskClient) Create(ctx context.Context, req *CreateTaskRequest) (*CreateTaskResponse, error) {
	var resp CreateTaskResponse
	if err := c.client.Call(ctx, "containerd.task.v2.Task", "Create", req, &resp); err != nil {
		return nil, err
	}
	return &resp, nil
}
```

`taskClient.Create` 调用 shim 的 `containerd.task.v2.Task.Create` ttrpc 创建 task。

##### containerd-shim-runc-v2 和 Runc

`containerd.task.v2.Task.Create` 是由 `containerd-shim-runc-v2` 提供的 ttrpc 服务。其服务实例是 `containerd-shim-runc-v2` 下的 `service` 对象：
```go
// containerd/cmd/containerd-shim-runc-v2/task/service.go
// Create a new initial process and container with the underlying OCI runtime
func (s *service) Create(ctx context.Context, r *taskAPI.CreateTaskRequest) (_ *taskAPI.CreateTaskResponse, err error) {
	...
	container, err := runc.NewContainer(ctx, s.platform, r)
	if err != nil {
		return nil, err
	}
	...
}
```

`service.Create` 调用 `runc.NewContainer` 实例化容器。`runc` 是实际创建容器的低级运行时。进入 `runc.NewContainer`：  
```go
// containerd/cmd/containerd-shim-runc-v2/runc/container.go
// NewContainer returns a new runc container
func NewContainer(ctx context.Context, platform stdio.Platform, r *task.CreateTaskRequest) (_ *Container, retErr error) {
	...
	// runc init 进程，runc init 进程负责启动容器进程
	p, err := newInit(
		ctx,
		r.Bundle,
		filepath.Join(r.Bundle, "work"),
		ns,
		platform,
		config,
		opts,
		rootfs,
	)

	...
	// 创建容器
	if err := p.Create(ctx, config); err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	container := &Container{
		ID:              r.ID,
		Bundle:          r.Bundle,
		process:         p,
		processes:       make(map[string]process.Process),
		reservedProcess: make(map[string]struct{}),
	}
	...
}
```

在 `Init.Create` 中调用低级运行时 `runc` 创建启动容器的 init 进程。  
```go
// containerd/cmd/containerd-shim-runc-v2/process/init.go
func (p *Init) Create(ctx context.Context, r *CreateConfig) error {
	...
	if err := p.runtime.Create(ctx, r.ID, r.Bundle, opts); err != nil {
		return p.runtimeError(err, "OCI runtime create failed")
	}
	...
}

// containerd/vendor/github.com/containerd/go-runc/runc.go
// Create creates a new container and returns its pid if it was created successfully
func (r *Runc) Create(context context.Context, id, bundle string, opts *CreateOpts) error {
	...
	cmd := r.command(context, append(args, id)...)
	if opts.IO != nil {
		opts.Set(cmd)
	}
	...
	// Runc.startCommand 执行 runc 命令创建容器
	ec, err := r.startCommand(cmd)
	...
}
```

在 `Runc.startCommand` 中执行 `runc init` 命令创建启动容器的 init 进程：
```bash
# ps -ef | grep 120376
root      4147233       1  0 14:25 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id nginx1 -address /run/containerd/containerd.sock
root      120396  4147233  0 14:25 ?        00:00:00 runc init
```
