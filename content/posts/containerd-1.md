+++
title = 'containerd 源码分析：创建 container（一）'
date = 2024-06-04T10:27:38+08:00
categories = ["kubernetes"]
tags = ["kubernetes", "containerd"]
author = 'xhy'
+++

# 前言

[Kubernetes:kubelet 源码分析之 pod 创建流程 ](https://www.cnblogs.com/xingzheanan/p/18202067) 介绍了 `kubelet` 创建 pod 的流程，[containerd 源码分析：kubelet 和 containerd 交互 
](https://www.cnblogs.com/xingzheanan/p/18206781) 介绍了 `kubelet` 通过 cri 接口和 `containerd` 交互的过程，[containerd 源码分析：启动注册流程 ](https://www.cnblogs.com/xingzheanan/p/18204637) 介绍了 `containerd` 作为高级容器运行时的启动流程。通过这三篇文章熟悉了 `kubelet` 和 `containerd` 的行为，对于 `containerd` 如何通过 OCI 接口创建容器 `container` 并没有涉及。

![kubelet 和 containerd 接口](/images/containerd.png)

本文将继续介绍 `containerd` 是如何创建容器 `container` 的。

# ctr

在介绍创建容器前，首先简单介绍下 `ctr`。`ctr` 是 `containerd` 的命令行客户端，本文会通过 `ctr` 进行调试和分析。

## ctr CLI

作为命令行工具 ctr 包括一系列和 `containerd` 交互的命令。主要命令如下：
```bash
COMMANDS:
   plugins, plugin            provides information about containerd plugins
   containers, c, container   manage containers
   images, image, i           manage images
   run                        run a container
   snapshots, snapshot        manage snapshots
   tasks, t, task             manage tasks
   install                    install a new package
   oci                        OCI tools
   shim                       interact with a shim directly
```

**containers|c|container**

不同与 Kubernetes 层面的 `container`，这里 `ctr` 命令管理的 `containers` 实际是管理存储在 [boltDB](https://github.com/boltdb/bolt) 中的 container metadata。 

创建 `container`：
```bash
# ctr c create docker.io/library/nginx:alpine nginx1
# ctr c ls
CONTAINER    IMAGE                             RUNTIME
nginx1       docker.io/library/nginx:alpine    io.containerd.runc.v2
```

通过 `boltbrowser` 查看 `boltDB` 存储的 container metadata，container metadata 存储在目录 `/var/lib/containerd/io.containerd.metadata.v1.bolt`。

![boltDB container](/images/boltDB-container.png)

**tasks|t|task**

task 是实际启动容器进程的命令，`ctr task start` 根据创建的 container 启动容器：
```bash
# ctr t start nginx1
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
...
```

**run**

ctr 的 run 命令，实际是 `ctr c create` 和 `ctr t start` 命令的组合。

接下来，使用 `ctr run` 命令做为调试参数分析完整的创建 container 容器的流程。

## ctr 调试

`ctr` 代码集中在 `containerd` 项目中，配置 `ctr` 的调试参数：
```json
{
   "version": "0.2.0",
   "configurations": [
      {
         "name": "ctr",
         "type": "go",
         "request": "launch",
         "mode": "auto",
         "program": "${fileDirname}",
         "args": ["run", "docker.io/library/nginx:alpine", "nginx1"]
      }
   ]
}
```

调试 `ctr`：  

![调试 ctr](/images/debug-ctr.png)

进入 `run.Command` 看其中做了什么。  
```go
// containerd/cmd/ctr/commands/run/run.go
// Command runs a container
var Command = &cli.Command{
	Name:      "run",
	Usage:     "Run a container",
   ...
   Action: func(context *cli.Context) error {
      ...
      // step1: 创建访问 containerd 的 client
      client, ctx, cancel, err := commands.NewClient(context)
		if err != nil {
			return err
		}
		defer cancel()

      // step2: 创建 container
      container, err := NewContainer(ctx, client, context)
		if err != nil {
			return err
		}
      ...

      opts := tasks.GetNewTaskOpts(context)
		ioOpts := []cio.Opt{cio.WithFIFODir(context.String("fifo-dir"))}
      // step3: 创建 task
		task, err := tasks.NewTask(ctx, client, container, context.String("checkpoint"), con, context.Bool("null-io"), context.String("log-uri"), ioOpts, opts...)
		if err != nil {
			return err
		}

      ...
      // step4: 启动 task
      if err := task.Start(ctx); err != nil {
			return err
		}
      ...
   }
}
```

在 `NewContainer` 中根据 `client` 创建 container。接着根据 container 创建 task，然后启动该 task 来启动容器。

### 创建 container

进入 `NewContainer`：  
```go
// containerd/cmd/ctr/commands/run/run_unix.go
func NewContainer(ctx gocontext.Context, client *containerd.Client, context *cli.Context) (containerd.Container, error) {
   ...
   return client.NewContainer(ctx, id, cOpts...)
}

// containerd/client/client.go
func (c *Client) NewContainer(ctx context.Context, id string, opts ...NewContainerOpts) (Container, error) {
   ...
   container := containers.Container{
		ID: id,
		Runtime: containers.RuntimeInfo{
			Name: c.runtime,
		},
	}
   ...
   // 调用 containerd 接口创建 container
   r, err := c.ContainerService().Create(ctx, container)
	if err != nil {
		return nil, err
	}
	return containerFromRecord(c, r), nil
}
```

重点在 `Client.ContainerService().Create`：  
```go
// containerd/client/containerstore.go
func (r *remoteContainers) Create(ctx context.Context, container containers.Container) (containers.Container, error) {
	created, err := r.client.Create(ctx, &containersapi.CreateContainerRequest{
		Container: containerToProto(&container),
	})
	if err != nil {
		return containers.Container{}, errdefs.FromGRPC(err)
	}

	return containerFromProto(created.Container), nil
}

// containerd/api/services/containers/v1/containers_grpc.pb.go
func (c *containersClient) Create(ctx context.Context, in *CreateContainerRequest, opts ...grpc.CallOption) (*CreateContainerResponse, error) {
	out := new(CreateContainerResponse)
	err := c.cc.Invoke(ctx, "/containerd.services.containers.v1.Containers/Create", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

调用 `/containerd.services.containers.v1.Containers/Create` grpc 接口创建 container。container 并不是容器进程，而是存储在数据库中的 container metadata。

`/containerd.services.containers.v1.Containers/Create` 是由 `containerd` 的 `io.containerd.grpc.v1.containers` 插件提供的服务： 
```go
// containerd/plugins/services/service.go
func (s *service) Create(ctx context.Context, req *api.CreateContainerRequest) (*api.CreateContainerResponse, error) {
	return s.local.Create(ctx, req)
}
```

插件实例调用 `local` 对象的 `Create` 方法创建 container。查看 `local` 对象具体指的什么。
```go
// containerd/plugins/services/service.go
func init() {
	registry.Register(&plugin.Registration{
		Type: plugins.GRPCPlugin,
		ID:   "containers",
		Requires: []plugin.Type{
			plugins.ServicePlugin,
		},
		InitFn: func(ic *plugin.InitContext) (interface{}, error) {
         // plugins.ServicePlugin：io.containerd.service.v1
         // services.ContainersService：containers-service
			i, err := ic.GetByID(plugins.ServicePlugin, services.ContainersService)
			if err != nil {
				return nil, err
			}
			return &service{local: i.(api.ContainersClient)}, nil
		},
	})
}
```

`local` 对象是 `containerd` 的 `io.containerd.service.v1.containers-service` 插件的实例。查看该实例的 `Create` 方法。  
```go
// containerd/plugins/services/containers/local.go
func (l *local) Create(ctx context.Context, req *api.CreateContainerRequest, _ ...grpc.CallOption) (*api.CreateContainerResponse, error) {
	var resp api.CreateContainerResponse

	if err := l.withStoreUpdate(ctx, func(ctx context.Context) error {
		container := containerFromProto(req.Container)

		created, err := l.Store.Create(ctx, container)
		if err != nil {
			return err
		}

		resp.Container = containerToProto(&created)

		return nil
	}); err != nil {
		return &resp, errdefs.ToGRPC(err)
	}
	...

	return &resp, nil
}
```

`local.Create` 调用 `local.withStoreUpdate` 方法创建 container。
```go
// containerd/plugins/services/containers/local.go
func (l *local) withStoreUpdate(ctx context.Context, fn func(ctx context.Context) error) error {
	return l.db.Update(l.withStore(ctx, fn))
}
```

`local.withStoreUpdate` 调用 `db` 对象的 `Update` 方法创建 container。
```go
// containerd/plugins/services/containers/local.go
func init() {
	registry.Register(&plugin.Registration{
		...
		InitFn: func(ic *plugin.InitContext) (interface{}, error) {
			m, err := ic.GetSingle(plugins.MetadataPlugin)
			if err != nil {
				return nil, err
			}
			ep, err := ic.GetSingle(plugins.EventPlugin)
			if err != nil {
				return nil, err
			}

			db := m.(*metadata.DB)
			return &local{
				Store:     metadata.NewContainerStore(db),
				db:        db,
				publisher: ep.(events.Publisher),
			}, nil
		},
	})
}
```

`db` 对象是 `io.containerd.metadata.v1` 插件的实例，该插件通过 `boltDB` 提供 metadata 存储服务。

metadata 插件实际调用的是匿名函数 fn 的内容，在 fn 中通过 `l.Store.Create(ctx, container)` 将 container 的 metadata 信息注册到 `boltDB` 数据库中。

创建 container 的过程实际是将 container 信息注册到 boltDB 的过程。
