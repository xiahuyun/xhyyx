+++
title = 'Kubernetes:kube-apiserver 之准入'
date = 2023-11-14T14:05:38+08:00
categories = ["kubernetes"]
tags = ["kubernetes", "kube-apiserver"]
author = 'xhy'
+++

# 前言

前两篇文章介绍了 `kube-apiserver` 的认证和鉴权，这里继续往下走，介绍 `kube-apiserver` 的准入。

# 准入 admission

不同于前两篇的逆序介绍，这里顺序介绍 `admission` 流程。从创建准入 `options`，到根据 `options` 创建准入 `config`，接着介绍在 `kube-apiserver` 的 `handler` 中是怎么进入准入控制，怎么执行的。

## admission options

进入 `NewOptions` 查看 `admission options` 是怎么创建的。
```go
# kubernetes/pkg/controlplane/apiserver/options/options.go
func NewOptions() *Options {
	s := Options{
        ...
		Admission:               kubeoptions.NewAdmissionOptions(),
    }
}

# kubernetes/pkg/kubeapiserver/options/admission.go
func NewAdmissionOptions() *AdmissionOptions {
	options := genericoptions.NewAdmissionOptions()
	// register all admission plugins
	RegisterAllAdmissionPlugins(options.Plugins)
	// set RecommendedPluginOrder
	options.RecommendedPluginOrder = AllOrderedPlugins
	// set DefaultOffPlugins
	options.DefaultOffPlugins = DefaultOffAdmissionPlugins()

	return &AdmissionOptions{
		GenericAdmission: options,
	}
}
```

`NewAdmissionOptions` 返回创建的 `AdmissionOptions`。其中，`options` 包括什么内容呢？我们看 `NewAdmissionOptions`， `RegisterAllAdmissionPlugins` 和 `DefaultOffAdmissionPlugins` 函数。

**`NewAdmissionOptions`**
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/server/options/admission.go
func NewAdmissionOptions() *AdmissionOptions {
	options := &AdmissionOptions{
        // 创建 Plugins 对象
		Plugins:    admission.NewPlugins(),
		Decorators: admission.Decorators{admission.DecoratorFunc(admissionmetrics.WithControllerMetrics)},
		RecommendedPluginOrder: []string{lifecycle.PluginName, mutatingwebhook.PluginName, validatingadmissionpolicy.PluginName, validatingwebhook.PluginName},
		DefaultOffPlugins:      sets.NewString(),
	}
    // 注册 admission plugins 到 Plugins 对象
	server.RegisterAllAdmissionPlugins(options.Plugins)
	return options
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
type Plugins struct {
	lock     sync.Mutex
	registry map[string]Factory
}

func NewPlugins() *Plugins {
	return &Plugins{}
}

# kubernetes/vendor/k8s.io/apiserver/pkg/server/plugins.go
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	lifecycle.Register(plugins)
	validatingwebhook.Register(plugins)
	mutatingwebhook.Register(plugins)
	validatingadmissionpolicy.Register(plugins)
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugin/namespace/lifecycle/admission.go
func Register(plugins *admission.Plugins) {
	plugins.Register(PluginName, func(config io.Reader) (admission.Interface, error) {
		return NewLifecycle(sets.NewString(metav1.NamespaceDefault, metav1.NamespaceSystem, metav1.NamespacePublic))
	})
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
func (ps *Plugins) Register(name string, plugin Factory) {
	...
	ps.registry[name] = plugin
}
```

在 `NewAdmissionOptions` 中，创建 `Plugins` 对象，将 `admission plugins` 注册到 `Plugins` 对象。注册的过程实际是写入 `admission plugin` 到对象 `registry` 的过程，`registry` 中存储的是 `admission plugin name` 和创建 `plugin` 工厂的映射。

**`RegisterAllAdmissionPlugins`**
```go
# kubernetes/pkg/kubeapiserver/options/plugins.go
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	admit.Register(plugins) // DEPRECATED as no real meaning
	alwayspullimages.Register(plugins)
    ...
}
```

类似于 `NewAdmissionOptions` 中的注册过程，`RegisterAllAdmissionPlugins` 将注册所有的 `admission plugin` 到 `Plugins` 对象中。注册之后的 `Plugins` 有 36 种 `admission plugin`。

**`DefaultOffAdmissionPlugins`**
```go
func DefaultOffAdmissionPlugins() sets.String {
	defaultOnPlugins := sets.NewString(
		lifecycle.PluginName,                    // NamespaceLifecycle
		limitranger.PluginName,                  // LimitRanger
        ...
	)

	return sets.NewString(AllOrderedPlugins...).Difference(defaultOnPlugins)
}
```

经过 `DefaultOffAdmissionPlugins` 处理后，`Plugins` 对象中有 20 种默认打开的 `admission plugin`，16 种默认关闭的 `admission plugin`。

## admission config

创建完 `admission options` 后开始创建 `admission config`。

```go
# kubernetes/cmd/kube-apiserver/app/server.go
func CreateKubeAPIServerConfig(opts options.CompletedOptions) (
	*controlplane.Config,
	aggregatorapiserver.ServiceResolver,
	[]admission.PluginInitializer,
	error,
) {
    err = opts.Admission.ApplyTo(
		genericConfig,
		versionedInformers,
		clientgoExternalClient,
		dynamicExternalClient,
		utilfeature.DefaultFeatureGate,
		pluginInitializers...)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to apply admission: %w", err)
	}
}

# kubernetes/pkg/kubeapiserver/options/admission.go
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeClient kubernetes.Interface,
	dynamicClient dynamic.Interface,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	if a == nil {
		return nil
	}

	if a.PluginNames != nil {
		// pass PluginNames to generic AdmissionOptions
		a.GenericAdmission.EnablePlugins, a.GenericAdmission.DisablePlugins = computePluginNames(a.PluginNames, a.GenericAdmission.RecommendedPluginOrder)
	}

	return a.GenericAdmission.ApplyTo(c, informers, kubeClient, dynamicClient, features, pluginInitializers...)
}

# kubernetes/vendor/k8s.io/apiserver/pkg/server/options/admission.go
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeClient kubernetes.Interface,
	dynamicClient dynamic.Interface,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	...
	admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
	if err != nil {
		return err
	}

	c.AdmissionControl = admissionmetrics.WithStepMetrics(admissionChain)
	return nil
}
```

经过多层调用到 `AdmissionOptions.ApplyTo` 方法，重点看其中的 `NewFromPlugins`。
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
func (ps *Plugins) NewFromPlugins(pluginNames []string, configProvider ConfigProvider, pluginInitializer PluginInitializer, decorator Decorator) (Interface, error) {
	handlers := []Interface{}
	mutationPlugins := []string{}
	validationPlugins := []string{}
    // 循环创建 plugin
	for _, pluginName := range pluginNames {
		...
        // 调用 Plugins.InitPlugin
		plugin, err := ps.InitPlugin(pluginName, pluginConfig, pluginInitializer)
		if err != nil {
			return nil, err
		}
		if plugin != nil {
			if decorator != nil {
				handlers = append(handlers, decorator.Decorate(plugin, pluginName))
			} else {
				handlers = append(handlers, plugin)
			}
            ...
		}
	}
	...
	return newReinvocationHandler(chainAdmissionHandler(handlers)), nil
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
func (ps *Plugins) InitPlugin(name string, config io.Reader, pluginInitializer PluginInitializer) (Interface, error) {
    // 调用 Plugins.getPlugin 创建 plugin
	plugin, found, err := ps.getPlugin(name, config)
	if err != nil {
		return nil, fmt.Errorf("couldn't init admission plugin %q: %v", name, err)
	}
	if !found {
		return nil, fmt.Errorf("unknown admission plugin: %s", name)
	}

	return plugin, nil
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
func (ps *Plugins) getPlugin(name string, config io.Reader) (Interface, bool, error) {
    ...
    // 通过 plugin 的 factory 创建 plugin
	ret, err := f(config2)
	return ret, true, err
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
type chainAdmissionHandler []Interface

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/reinvocation.go
func newReinvocationHandler(admissionChain Interface) Interface {
	return &reinvoker{admissionChain}
}

type reinvoker struct {
	admissionChain Interface
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/interfaces.go
type Interface interface {
	// Handles returns true if this admission controller can handle the given operation
	// where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
	Handles(operation Operation) bool
}
```

`NewFromPlugins` 主要做了三件事：
- 根据 `plugin name`，通过 `plugin factory` 循环创建 `plugin`。
- 将 `plugin` 添加到 `handlers`，并且转换为 `chainAdmissionHandler` 数组，数组中存储的是实现接口 `Interface` 的实例。
- 将 `chainAdmissionHandler` 赋给 `reinvoker`。

## admission plugin

前面两节介绍了 `admission options` 和 `admission config`。在继续往下介绍之前，有必要介绍 `admission plugin`。

`admission plugin` 类型分为变更 `plugin` 和验证 `plugin`，分别实现了 `MutationInterface` 和 `ValidationInterface` 接口。
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/admission/interfaces.go
type Interface interface {
	// Handles returns true if this admission controller can handle the given operation
	// where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
	Handles(operation Operation) bool
}

type MutationInterface interface {
	Interface

	// Admit makes an admission decision based on the request attributes.
	// Context is used only for timeout/deadline/cancellation and tracing information.
	Admit(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}

// ValidationInterface is an abstract, pluggable interface for Admission Control decisions.
type ValidationInterface interface {
	Interface

	// Validate makes an admission decision based on the request attributes.  It is NOT allowed to mutate
	// Context is used only for timeout/deadline/cancellation and tracing information.
	Validate(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}
```

`MutationInterface` 和 `ValidationInterface` 都包括 `Interface` 接口，实现变更和验证的 `plugin` 也要实现 `Interface` 的 `Handlers` 方法。

以 `AlwaysPullImages plugin` 为例，查看其实现的方法。
```go
# kubernetes/plugin/pkg/admission/alwaypullimages/admission.go
type AlwaysPullImages struct {
	*admission.Handler
}

func (a *AlwaysPullImages) Admit(ctx context.Context, attributes admission.Attributes, o admission.ObjectInterfaces) (err error) {
	...
}

func (*AlwaysPullImages) Validate(ctx context.Context, attributes admission.Attributes, o admission.ObjectInterfaces) (err error) {
	...
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/handler.go
// AlwaysPullImages 和 Handler 是组合关系
// AlwaysPullImages 实现了 Handlers 方法
type Handler struct {
	operations sets.String
	readyFunc  ReadyFunc
}

// Handles returns true for methods that this handler supports
func (h *Handler) Handles(operation Operation) bool {
	return h.operations.Has(string(operation))
}
```

可以看到，`AlwaysPullImages plugin` 既是变更 `plugin` 也是验证 `plugin`。

那么，`plugin` 的变更和验证是什么时候调用的呢。继续往下看。

## admission handler

`admission handler` 实际上是一段嵌在 `RESTful API handler` 的代码，这段代码作用在 `CREATE`，`POST`，`DELETE` action 上，对于 `GET` action，不需要做变更和验证操作。

查看 `admission handler`。
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	admit := a.group.Admit
	...
	for _, action := range actions {
		switch action.Verb {
		case "POST": // Create a resource.
			var handler restful.RouteFunction
			if isNamedCreater {
				handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
			} else {
				handler = restfulCreateResource(creater, reqScope, admit)
			}
			...
			route := ws.POST(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("create"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				// TODO: in some cases, the API may return a v1.Status instead of the versioned object
				// but currently go-restful can't handle multiple different objects being returned.
				Returns(http.StatusCreated, "Created", producedObject).
				Returns(http.StatusAccepted, "Accepted", producedObject).
				Reads(defaultVersionedObject).
				Writes(producedObject)
				...
		}
	}
}
```

这里以 `POST` action 为例，查看 `RESTful API handler` 是怎么做准入控制的。

进入 `restfulCreateResource`（`restfulCreateNamedResource` 类似）查看 `handler` 的创建过程。  
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func restfulCreateResource(r rest.Creater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.CreateResource(r, &scope, admit)(res.ResponseWriter, req.Request)
	}
}

# kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/create.go
// CreateResource returns a function that will handle a resource creation.
func CreateResource(r rest.Creater, scope *RequestScope, admission admission.Interface) http.HandlerFunc {
	return createHandler(&namedCreaterAdapter{r}, scope, admission, false)
}

func createHandler(r rest.NamedCreater, scope *RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		admit = admission.WithAudit(admit)
		// 获得请求的 attributes，该 attributes 会送入准入控制中
		admissionAttributes := admission.NewAttributesRecord(obj, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Create, options, dryrun.IsDryRun(options.DryRun), userInfo)
		requestFunc := func() (runtime.Object, error) {
			return r.Create(
				ctx,
				name,
				obj,
				// 返回验证准入 attributes 的函数
				rest.AdmissionToValidateObjectFunc(admit, admissionAttributes, scope),
				options,
			)
		}

		result, err := finisher.FinishRequest(ctx, func() (runtime.Object, error) {
			...
			// 判断 admit 是否实现了变更接口，如果实现了，执行变更方法
			if mutatingAdmission, ok := admit.(admission.MutationInterface); ok && mutatingAdmission.Handles(admission.Create) {
				if err := mutatingAdmission.Admit(ctx, admissionAttributes, scope); err != nil {
					return nil, err
				}
			}
			...
			result, err := requestFunc()

			return result, err
		})
	}
}
```

`requestFunc` 负责和 `etcd` 交互以创建资源，它是一个函数，调用点在变更 `plugin` 之后。对请求的执行顺序是，先执行变更准入，再执行验证准入。

分别看变更和验证准入的调用。

### 变更准入
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/create.go
admit = admission.WithAudit(admit)
...

result, err := finisher.FinishRequest(ctx, func() (runtime.Object, error) {
	admit = fieldmanager.NewManagedFieldsValidatingAdmissionController(admit)
	if mutatingAdmission, ok := admit.(admission.MutationInterface); ok && mutatingAdmission.Handles(admission.Create) {
		if err := mutatingAdmission.Admit(ctx, admissionAttributes, scope); err != nil {
			return nil, err
		}
	}
})

# kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/fieldmanager/admission.go
func NewManagedFieldsValidatingAdmissionController(wrap admission.Interface) admission.Interface {
	if wrap == nil {
		return nil
	}
	return &managedFieldsValidatingAdmissionController{wrap: wrap}
}

func (admit *managedFieldsValidatingAdmissionController) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) (err error) {
	mutationInterface, isMutationInterface := admit.wrap.(admission.MutationInterface)
	if !isMutationInterface {
		return nil
	}
	...
	objectMeta, err := meta.Accessor(a.GetObject())
	...

	managedFieldsBeforeAdmission := objectMeta.GetManagedFields()
	if err := mutationInterface.Admit(ctx, a, o); err != nil {
		return err
	}
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/audit.go
func WithAudit(i Interface) Interface {
	if i == nil {
		return i
	}
	return &auditHandler{Interface: i}
}

func (handler *auditHandler) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
	if !handler.Interface.Handles(a.GetOperation()) {
		return nil
	}
	...
	var err error
	if mutator, ok := handler.Interface.(MutationInterface); ok {
		err = mutator.Admit(ctx, a, o)
		handler.logAnnotations(ctx, a)
	}
	return err
}
```

可以看到 `mutatingAdmission.Admit` 的调用链是从 `managedFieldsValidatingAdmissionController` 到 `auditHandler`。最终执行到 `admission config` 中创建的 `AdmissionControl`。
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/server/options/admission.go
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeClient kubernetes.Interface,
	dynamicClient dynamic.Interface,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	...
	admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
	if err != nil {
		return err
	}

	c.AdmissionControl = admissionmetrics.WithStepMetrics(admissionChain)
}
```

继续查看 `AdmissionControl` 的 `Admit` 方法。
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/admission/metrics/metrics.go
func WithStepMetrics(i admission.Interface) admission.Interface {
	return WithMetrics(i, Metrics.ObserveAdmissionStep)
}

// WithMetrics is a decorator for admission handlers with a generic observer func.
func WithMetrics(i admission.Interface, observer ObserverFunc, extraLabels ...string) admission.Interface {
	return &pluginHandlerWithMetrics{
		Interface:   i,
		observer:    observer,
		extraLabels: extraLabels,
	}
}

func (p pluginHandlerWithMetrics) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
	mutatingHandler, ok := p.Interface.(admission.MutationInterface)
	if !ok {
		return nil
	}

	start := time.Now()
	err := mutatingHandler.Admit(ctx, a, o)
	...
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
func (ps *Plugins) NewFromPlugins(pluginNames []string, configProvider ConfigProvider, pluginInitializer PluginInitializer, decorator Decorator) (Interface, error) {
	...
	return newReinvocationHandler(chainAdmissionHandler(handlers)), nil
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/reinvocation.go
func newReinvocationHandler(admissionChain Interface) Interface {
	return &reinvoker{admissionChain}
}

func (r *reinvoker) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
	if mutator, ok := r.admissionChain.(MutationInterface); ok {
		err := mutator.Admit(ctx, a, o)
		if err != nil {
			return err
		}
		...
	}
	return nil
}

# kubernetes/vendor/k8s.io/apiserver/pkg/admission/chain.go
type chainAdmissionHandler []Interface

func (admissionHandler chainAdmissionHandler) Admit(ctx context.Context, a Attributes, o ObjectInterfaces) error {
	for _, handler := range admissionHandler {
		if !handler.Handles(a.GetOperation()) {
			continue
		}
		if mutator, ok := handler.(MutationInterface); ok {
			err := mutator.Admit(ctx, a, o)
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```

通过接口实例的逐层调用，最终执行到 `chainAdmissionHandler` 的 `Admit` 方法。在该方法内，遍历 `handler`。首先执行 `handler` 的 `Handler` 方法，查看是否支持 `RESTful API action` 的变更操作。 如果支持执行 `handler` 的 `Admit` 方法。如果不支持，执行下一个 `handler`。

`handler` 的 `Admit` 实际执行的是 `plugin.Admit`。以 `AlwaysPullImages plugin` 为例查看其 `Admit` 变更准入过程。
```go
# kubernetes/plugin/pkg/admission/alwayspullimages/admission.go
func (a *AlwaysPullImages) Admit(ctx context.Context, attributes admission.Attributes, o admission.ObjectInterfaces) (err error) {
	// Ignore all calls to subresources or resources other than pods.
	if shouldIgnore(attributes) {
		return nil
	}
	pod, ok := attributes.GetObject().(*api.Pod)
	if !ok {
		return apierrors.NewBadRequest("Resource was marked with kind Pod but was unable to be converted")
	}

	pods.VisitContainersWithPath(&pod.Spec, field.NewPath("spec"), func(c *api.Container, _ *field.Path) bool {
		c.ImagePullPolicy = api.PullAlways
		return true
	})

	return nil
}
```

可以看到在 `VisitContainersWithPath` 中，将 `container` 的 `imagePullPolicy` 更新为 `Always`，从而实现变更准入。

### 验证准入

查看 `RESTful API handler` 的验证准入过程。
```go
# kubernetes/vendor/k8s.io/apiserver/endpoints/handlers/create.go
func createHandler(r rest.NamedCreater, scope *RequestScope, admit admission.Interface, includeName bool) http.HandlerFunc {
	return func(w http.ResponseWriter, req *http.Request) {
		requestFunc := func() (runtime.Object, error) {
			return r.Create(
				ctx,
				name,
				obj,
				rest.AdmissionToValidateObjectFunc(admit, admissionAttributes, scope),
				options,
			)
		}

		result, err := finisher.FinishRequest(ctx, func() (runtime.Object, error) {
			if mutatingAdmission, ok := admit.(admission.MutationInterface); ok && mutatingAdmission.Handles(admission.Create) {
				if err := mutatingAdmission.Admit(ctx, admissionAttributes, scope); err != nil {
					return nil, err
				}
			}

			result, err := requestFunc()
			return result, err
		})
	}
}
```

变更准入成功后，开始执行验证准入。验证准入的逻辑定义在 `AdmissionToValidateObjectFunc`，资源实体 `r` 和 `etcd` 交互时，首先进行验证准入：
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go
func (e *Store) Create(ctx context.Context, obj runtime.Object, createValidation rest.ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error) {
	if createValidation != nil {
		// 执行验证准入
		if err := createValidation(ctx, obj.DeepCopyObject()); err != nil {
			return nil, err
		}
	}

	// 验证准入成功后开始和 etcd 交互
	name, err := e.ObjectNameFunc(obj)
	if err != nil {
		return nil, err
	}
	key, err := e.KeyFunc(ctx, name)
	if err != nil {
		return nil, err
	}
	...
}
```

知道了验证准入的流程。我们看验证准入具体做了什么。
```go
# kubernetes/vendor/k8s.io/apiserver/pkg/registry/rest/create.go
func AdmissionToValidateObjectFunc(admit admission.Interface, staticAttributes admission.Attributes, o admission.ObjectInterfaces) ValidateObjectFunc {
	validatingAdmission, ok := admit.(admission.ValidationInterface)
	if !ok {
		return func(ctx context.Context, obj runtime.Object) error { return nil }
	}
	return func(ctx context.Context, obj runtime.Object) error {
		name := staticAttributes.GetName()
		...

		finalAttributes := admission.NewAttributesRecord(
			obj,
			staticAttributes.GetOldObject(),
			staticAttributes.GetKind(),
			staticAttributes.GetNamespace(),
			name,
			staticAttributes.GetResource(),
			staticAttributes.GetSubresource(),
			staticAttributes.GetOperation(),
			staticAttributes.GetOperationOptions(),
			staticAttributes.IsDryRun(),
			staticAttributes.GetUserInfo(),
		)
		if !validatingAdmission.Handles(finalAttributes.GetOperation()) {
			return nil
		}
		return validatingAdmission.Validate(ctx, finalAttributes, o)
	}
}
```

类似于变更准入，首先调用 `Handlers` 查看 `plugin` 是否支持 `RESTful API` 请求的操作。如果支持调用 `Validate` 进行验证准入。

验证准入的调用过程和变更准入非常类似，这里不过多介绍了。最终，经过层层调用执行 `plugin` 的验证准入。这里，以 `AlwaysPullImages plugin` 为例，查看验证准入过程。
```go
# kubernetes/plugin/pkg/admission/alwayspullimages/admission.go
func (*AlwaysPullImages) Validate(ctx context.Context, attributes admission.Attributes, o admission.ObjectInterfaces) (err error) {
	...
	pod, ok := attributes.GetObject().(*api.Pod)
	if !ok {
		return apierrors.NewBadRequest("Resource was marked with kind Pod but was unable to be converted")
	}

	var allErrs []error
	pods.VisitContainersWithPath(&pod.Spec, field.NewPath("spec"), func(c *api.Container, p *field.Path) bool {
		if c.ImagePullPolicy != api.PullAlways {
			allErrs = append(allErrs, admission.NewForbidden(attributes,
				field.NotSupported(p.Child("imagePullPolicy"), c.ImagePullPolicy, []string{string(api.PullAlways)}),
			))
		}
		return true
	})
	if len(allErrs) > 0 {
		return utilerrors.NewAggregate(allErrs)
	}

	return nil
}
```

可以看到，`AlwaysPullImages plugin` 验证 `container` 的 `imagePullPolicy` 是否是 `Always`。

# 小结

通过本篇文章介绍了 `kube-apiserver` 中的 `admission` 准入流程。美好的时光总是短暂的，关于 `kube-apiserver` 的介绍基本结束了。下面开始 `kube-scheduler` 的介绍，敬请期待。
