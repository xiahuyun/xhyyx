+++
title = 'Kubernetes:kubelet 源码分析之探针'
date = 2024-05-20T15:23:38+08:00
categories = ["kubernetes"]
tags = ["kubernetes", "kubelet", "probe"]
author = 'xhy'
+++

# 前言

`kubernetes` 提供三种探针，配置探针（Liveness），就绪探针（Readiness）和启动（Startup）探针判断容器健康状态。其中，存活探针确定什么时候重启容器，就绪探针确定容器何时准备好接受流量请求，启动探针判断应用容器何时启动。

本文通过分析 `kubelet` 源码了解 `kubernetes` 的探针是怎么工作的。 

# kubelet probeManager

`kubelet` 中的 `probeManager` 模块提供了探针服务，直接分析 `probeManager`。

```go
// kubernetes/pkg/kubelet/kubelet.go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,...) (*Kubelet, error) {
    ...
    klet.livenessManager = proberesults.NewManager()
	klet.readinessManager = proberesults.NewManager()
	klet.startupManager = proberesults.NewManager()

    ...
	if kubeDeps.ProbeManager != nil {
		klet.probeManager = kubeDeps.ProbeManager
	} else {
		klet.probeManager = prober.NewManager(
			klet.statusManager,
			klet.livenessManager,
			klet.readinessManager,
			klet.startupManager,
			klet.runner,
			kubeDeps.Recorder)
	}
    ...
}
```

在 `NewMainKubelet` 中初始化 `probeManager`。其中，`probeManager` 包括三种探针 `statusManager`，`livenessManager` 和 `readinessManager`。

当 `kubelet` 处理 pod 时，会将 pod 添加到 `probeManager`：
```go
// kubernetes/pkg/kubelet/kubelet.go
func (kl *Kubelet) SyncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
	...
	// Ensure the pod is being probed
	kl.probeManager.AddPod(pod)
    ...
}

// kubernetes/pkg/kubelet/prober/prober_manager.go
func (m *manager) AddPod(pod *v1.Pod) {
    ...
    key := probeKey{podUID: pod.UID}
    for _, c := range append(pod.Spec.Containers, getRestartableInitContainers(pod)...) {
        key.containerName = c.Name

		if c.StartupProbe != nil {
			...
		}

        if c.ReadinessProbe != nil {
			key.probeType = readiness
			if _, ok := m.workers[key]; ok {
				klog.V(8).ErrorS(nil, "Readiness probe already exists for container",
					"pod", klog.KObj(pod), "containerName", c.Name)
				return
			}
			w := newWorker(m, readiness, pod, c)
			m.workers[key] = w
			go w.run()
		}

        if c.LivenessProbe != nil {
			...
		}
    }
}
```

在 `manager.AddPod` 中包含三种探针的处理逻辑，这里以 `ReadinessProbe` 探针为例进行分析。首先，创建 `ReadinessProbe` 的 worker，接着开启一个协程运行该 worker：
```go
// kubernetes/pkg/kubelet/prober/worker.go
func (w *worker) run() {
	...
probeLoop:
    // doProbe 进行探针检测
	for w.doProbe(ctx) {
		// Wait for next probe tick.
		select {
		case <-w.stopCh:
			break probeLoop
		case <-probeTicker.C:
		case <-w.manualTriggerCh:
			// continue
		}
	}
}

func (w *worker) doProbe(ctx context.Context) (keepGoing bool) {
    ...
    // Note, exec probe does NOT have access to pod environment variables or downward API
	result, err := w.probeManager.prober.probe(ctx, w.probeType, w.pod, status, w.container, w.containerID)
	if err != nil {
		// Prober error, throw away the result.
		return true
	}
    ...
}
```

进入 `worker.probeManager.prober.probe` 查看探针是怎么探测 container 的：
```go
// kubernetes/pkg/kubelet/prober/prober.go
// probe probes the container.
func (pb *prober) probe(ctx context.Context, probeType probeType, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID) (results.Result, error) {
	var probeSpec *v1.Probe
	switch probeType {
	case readiness:
		probeSpec = container.ReadinessProbe
	case liveness:
		probeSpec = container.LivenessProbe
	case startup:
		probeSpec = container.StartupProbe
	default:
		return results.Failure, fmt.Errorf("unknown probe type: %q", probeType)
	}

    if probeSpec == nil {
		klog.InfoS("Probe is nil", "probeType", probeType, "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name)
		return results.Success, nil
	}

    result, output, err := pb.runProbeWithRetries(ctx, probeType, probeSpec, pod, status, container, containerID, maxProbeRetries)
    ...
}

// runProbeWithRetries tries to probe the container in a finite loop, it returns the last result
// if it never succeeds.
func (pb *prober) runProbeWithRetries(ctx context.Context, probeType probeType, p *v1.Probe, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID, retries int) (probe.Result, string, error) {
	var err error
	var result probe.Result
	var output string
	for i := 0; i < retries; i++ {
		result, output, err = pb.runProbe(ctx, probeType, p, pod, status, container, containerID)
		if err == nil {
			return result, output, nil
		}
	}
	return result, output, err
}

func (pb *prober) runProbe(ctx context.Context, probeType probeType, p *v1.Probe, pod *v1.Pod, status v1.PodStatus, container v1.Container, containerID kubecontainer.ContainerID) (probe.Result, string, error) {
	timeout := time.Duration(p.TimeoutSeconds) * time.Second
	if p.Exec != nil {
        klog.V(4).InfoS("Exec-Probe runProbe", "pod", klog.KObj(pod), "containerName", container.Name, "execCommand", p.Exec.Command)
		command := kubecontainer.ExpandContainerCommandOnlyStatic(p.Exec.Command, container.Env)
		return pb.exec.Probe(pb.newExecInContainer(ctx, container, containerID, command, timeout))
    }

    if p.HTTPGet != nil {
        req, err := httpprobe.NewRequestForHTTPGetAction(p.HTTPGet, &container, status.PodIP, "probe")
        ...
    }

    if p.TCPSocket != nil {
        ...
    }

    if p.GRPC != nil {
        ...
    }
    ...
}
```

到这里我们可以看到，根据探针的不同类型执行不同的方法，对于用命令行探测的探针，执行 `prober.exec.Probe` 方法，对于 http 类型的探针，执行 `httpprobe.NewRequestForHTTPGetAction` 类型的方法，等等。

# 小结

本文从 `kubelet` 源码层面介绍了 `kubernetes` 中探针的检测逻辑，力图做到知其然，知其所以然。
