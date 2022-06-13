#### 1.观察方式
调试scheduler 调度时间指标
prometheus 调度时间指标 调度到bound总时间
```
histogram_quantile(0.99, sum(rate(scheduler_e2e_scheduling_duration_seconds_bucket{job="kube-scheduler-discovery"}[5m])) without(instance, pod))
```
prometheus 调度时间指标 算法打分时间
```
histogram_quantile(0.99, sum(rate(scheduler_scheduling_algorithm_duration_seconds_bucket{job="kube-scheduler-discovery"}[5m])) by (le))
```
prometheus 调度时间指标 具体每个阶段时间
```
histogram_quantile(0.999, sum by(extension_point,le) (rate(scheduler_framework_extension_point_duration_seconds_bucket{job="kube-scheduler-discovery"}[5m])))
```


prometheus  ingress
1.20
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prom
  namespace: openstack
spec:
  rules:
  - host: prom35.local
    http:
      paths:
      - backend:
          service:
            name: prom-metrics
            port:
              number: 9090
        path: /
        pathType: ImplementationSpecific
```
1.16
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prom
  namespace: openstack
spec:
  rules:
  - host: prom35.local
    http:
      paths:
      - backend:
          serviceName: prom-metrics
          servicePort: http
        path: /
```
[scheduler更多指标](https://segmentfault.com/a/1190000041670250)

#### 2.性能优化
优化点
1. PreFilter 接口需要在nodeaffinity中实现 并且在PreFilter阶段对pod进行一次对应
pod.Spec.Affinity的解析并写入`cycleState.Write(preFilterStateKey, state)`
现阶段nodeaffinity score会对每个节点均进行一次pod.Spec.Affinity解析 在大规模场景下
影响效率
```
		PreFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.FitName},
				{Name: nodeports.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
				{Name: volumebinding.Name},
			},
		},
```
2.PreFilter 由于daemonset实际能在PreFilter阶段直接筛选出对应节点
```
func (f *frameworkImpl) RunPreFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preFilterPlugins {
		status = f.runPreFilterPlugin(ctx, pl, state, pod)
		if !status.IsSuccess() {
			if status.IsUnschedulable() {
				return status
			}
			err := status.AsError()
			klog.ErrorS(err, "Failed running PreFilter plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running PreFilter plugin %q: %w", pl.Name(), err))
		}
	}

	return nil
}
这里从runPreFilterPlugin 插件列表处理原始的节点信息及pod列表等 实际当能在o(1)的时间复杂度内调度时 此PreFilter过滤器可以直接给出建议 
```
```
// 实际的主要处理主要是对daemonset pod中的
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchFields:
          - key: metadata.name
            operator: In
            values:
            - node-6
// 此标签的取值 并在PreFilter 阶段返回，此nodeAffinity 添加者为daemonset控制器
// PreFilter builds and writes cycle state used by Filter.
func (pl *NodeAffinity) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) (*framework.PreFilterResult, *framework.Status) {
	state := &preFilterState{requiredNodeSelectorAndAffinity: nodeaffinity.GetRequiredNodeAffinity(pod)}
	cycleState.Write(preFilterStateKey, state)
	affinity := pod.Spec.Affinity
	if affinity == nil ||
		affinity.NodeAffinity == nil ||
		affinity.NodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution == nil ||
		len(affinity.NodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution.NodeSelectorTerms) == 0 {
		return nil, nil
	}

	// Check if there is affinity to a specific node and return it.
	terms := affinity.NodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution.NodeSelectorTerms
	var nodeNames sets.String
	for _, t := range terms {
		var termNodeNames sets.String
		for _, r := range t.MatchFields {
			if r.Key == metav1.ObjectNameField && r.Operator == v1.NodeSelectorOpIn {
				// The requirements represent ANDed constraints, and so we need to
				// find the intersection of nodes.
				s := sets.NewString(r.Values...)
				if termNodeNames == nil {
					termNodeNames = s
				} else {
					termNodeNames = termNodeNames.Intersection(s)
				}
			}
		}
		if termNodeNames == nil {
			// If this term has no node.Name field affinity,
			// then all nodes are eligible because the terms are ORed.
			return nil, nil
		}
		// If the set is empty, it means the terms had affinity to different
		// sets of nodes, and since they are ANDed, then the pod will not match any node.
		if len(termNodeNames) == 0 {
			return nil, framework.NewStatus(framework.UnschedulableAndUnresolvable, errReasonConflict)
		}
		nodeNames = nodeNames.Union(termNodeNames)
	}
	if nodeNames != nil {
		return &framework.PreFilterResult{NodeNames: nodeNames}, nil
	}
	return nil, nil

}
```

kubernetes/pkg/controller/daemon/daemon_controller.go
```
// daemonset控制器 添加nodeAffinity相关逻辑主要代码
klog.V(4).Infof("Nodes needing daemon pods for daemon set %s: %+v, creating %d", ds.Name, nodesNeedingDaemonPods, createDiff)
......
		for i := pos; i < pos+batchSize; i++ {
			go func(ix int) {
				defer createWait.Done()

				podTemplate := template.DeepCopy()
// 这里对daemonset pod的NodeAffinity进行了替换
	  util.ReplaceDaemonSetPodNodeNameNodeAffinity进行了替换(
					podTemplate.Spec.Affinity, nodesNeedingDaemonPods[ix])

				err := dsc.podControl.CreatePods(ctx, ds.Namespace, podTemplate,
					ds, metav1.NewControllerRef(ds, controllerKind))

				if err != nil {
					if apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
						// If the namespace is being torn down, we can safely ignore
						// this error since all subsequent creations will fail.
						return
					}
				}
				if err != nil {
					klog.V(2).Infof("Failed creation, decrementing expectations for set %q/%q", ds.Namespace, ds.Name)
					dsc.expectations.CreationObserved(dsKey)
					errCh <- err
					utilruntime.HandleError(err)
				}
			}(i)
		}
......
```
因此调度器在默认的情况下 应该尽快处理daemonset对应的pod 

#### 3.调度阶段及对应执行过滤器(代码分析)
scheduler启动中默认注册的插件
```
apiVersion: kubescheduler.config.k8s.io/v1beta1
clientConnection:
  acceptContentTypes: ""
  burst: 100
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-scheduler-kubeconfig.yaml
  qps: 50
enableContentionProfiling: true
enableProfiling: false
healthzBindAddress: 0.0.0.0:10251
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: true
  leaseDuration: 15s
  renewDeadline: 10s
  resourceLock: leases
  resourceName: kube-scheduler
  resourceNamespace: kube-system
  retryPeriod: 2s
metricsBindAddress: 0.0.0.0:10251
parallelism: 16
percentageOfNodesToScore: 0
podInitialBackoffSeconds: 1
podMaxBackoffSeconds: 10
profiles:
- pluginConfig:
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      kind: DefaultPreemptionArgs
      minCandidateNodesAbsolute: 100
      minCandidateNodesPercentage: 10
    name: DefaultPreemption
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      hardPodAffinityWeight: 1
      kind: InterPodAffinityArgs
    name: InterPodAffinity
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      kind: NodeAffinityArgs
    name: NodeAffinity
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      kind: NodeResourcesFitArgs
    name: NodeResourcesFit
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      kind: NodeResourcesLeastAllocatedArgs
      resources:
      - name: cpu
        weight: 1
      - name: memory
        weight: 1
    name: NodeResourcesLeastAllocated
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      defaultingType: System
      kind: PodTopologySpreadArgs
    name: PodTopologySpread
  - args:
      apiVersion: kubescheduler.config.k8s.io/v1beta1
      bindTimeoutSeconds: 600
      kind: VolumeBindingArgs
    name: VolumeBinding
  plugins:
    bind:
      enabled:
      - name: DefaultBinder
        weight: 0
    filter:
      enabled:
      - name: NodeUnschedulable
        weight: 0
      - name: NodeName
        weight: 0
      - name: TaintToleration
        weight: 0
      - name: NodeAffinity
        weight: 0
      - name: NodePorts
        weight: 0
      - name: NodeResourcesFit
        weight: 0
      - name: VolumeRestrictions
        weight: 0
      - name: EBSLimits
        weight: 0
      - name: GCEPDLimits
        weight: 0
      - name: NodeVolumeLimits
        weight: 0
      - name: AzureDiskLimits
        weight: 0
      - name: VolumeBinding
        weight: 0
      - name: VolumeZone
        weight: 0
      - name: PodTopologySpread
        weight: 0
      - name: InterPodAffinity
        weight: 0
    permit: {}
    postBind: {}
    postFilter:
      enabled:
      - name: DefaultPreemption
        weight: 0
    preBind:
      enabled:
      - name: VolumeBinding
        weight: 0
    preFilter:
      enabled:
      - name: NodeResourcesFit
        weight: 0
      - name: NodePorts
        weight: 0
      - name: PodTopologySpread
        weight: 0
      - name: InterPodAffinity
        weight: 0
      - name: VolumeBinding
        weight: 0
    preScore:
      enabled:
      - name: InterPodAffinity
        weight: 0
      - name: PodTopologySpread
        weight: 0
      - name: TaintToleration
        weight: 0
    queueSort:
      enabled:
      - name: PrioritySort
        weight: 0
    reserve:
      enabled:
      - name: VolumeBinding
        weight: 0
    score:
      enabled:
      - name: NodeResourcesBalancedAllocation
        weight: 1
      - name: ImageLocality
        weight: 1
      - name: InterPodAffinity
        weight: 1
      - name: NodeResourcesLeastAllocated
        weight: 1
      - name: NodeAffinity
        weight: 1
      - name: NodePreferAvoidPods
        weight: 10000
      - name: PodTopologySpread
        weight: 2
      - name: TaintToleration
        weight: 1
  schedulerName: default-scheduler
```
此配置列举了scheduler应用到的几乎所有配置 因此以阶段分别介绍阶段中应用的每个插件
的作用及主要代码介绍 这里主要介绍默认情况下enable的所有插件
![scheduler.svg](../_resources/scheduler.svg)

如何初始化并进入
- 1.scheduler setup
```
// 设置scheduler实例
// Setup creates a completed config and a scheduler based on the command args and options
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	if errs := opts.Validate(); len(errs) > 0 {
		return nil, nil, utilerrors.NewAggregate(errs)
	}

	c, err := opts.Config()
	if err != nil {
		return nil, nil, err
	}

	// Get the completed config
	cc := c.Complete()

	outOfTreeRegistry := make(runtime.Registry)
	for _, option := range outOfTreeRegistryOptions {
		if err := option(outOfTreeRegistry); err != nil {
			return nil, nil, err
		}
	}

	recorderFactory := getRecorderFactory(&cc)
	completedProfiles := make([]kubeschedulerconfig.KubeSchedulerProfile, 0)
	// Create the scheduler.
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		recorderFactory,
		ctx.Done(),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
		scheduler.WithParallelism(cc.ComponentConfig.Parallelism),
		scheduler.WithBuildFrameworkCapturer(func(profile kubeschedulerconfig.KubeSchedulerProfile) {
			// Profiles are processed during Framework instantiation to set default plugins and configurations. Capturing them for logging
			completedProfiles = append(completedProfiles, profile)
		}),
	)
	if err != nil {
		return nil, nil, err
	}
	if err := options.LogOrWriteConfig(opts.WriteConfigTo, &cc.ComponentConfig, completedProfiles); err != nil {
		return nil, nil, err
	}

	return &cc, sched, nil
}
```
进入scheduler.New
```
// New returns a Scheduler
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

	schedulerCache := internalcache.New(30*time.Second, stopEverything)
	
	// 内部插件初始化对象 会调用插件实现对象的New方法对插件对象struct初始化
	// 插件合并 外置插件会从main函数进行传递 通常 需要从main函数传递 因此需要
	// 重新编译scheduler 实现的方法是在 ../kubernetes/cmd/kube-   
	// scheduler/scheduler.go中 向app.NewSchedulerCommand()
	// 用app.WithPlugin传递自己的插件及对应的New方法

	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	snapshot := internalcache.NewEmptySnapshot()

	configurator := &Configurator{
		client:                   client,
		recorderFactory:          recorderFactory,
		informerFactory:          informerFactory,
		schedulerCache:           schedulerCache,
		StopEverything:           stopEverything,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		podInitialBackoffSeconds: options.podInitialBackoffSeconds,
		podMaxBackoffSeconds:     options.podMaxBackoffSeconds,
		// 配置profiles profiles可以由外部指定 其中包含了各种插件的启用
		// 可以配置多个 示例如下
		profiles:                 append([]schedulerapi.KubeSchedulerProfile(nil), options.profiles...),
		// 插件配置 插件分为InTreeRegistry插件和OutOfTreeRegistry此插件 这里
		// 是合并之后的结果
		registry:                 registry,
		nodeInfoSnapshot:         snapshot,
		extenders:                options.extenders,
		frameworkCapturer:        options.frameworkCapturer,
	}

	metrics.Register()

	var sched *Scheduler
	source := options.schedulerAlgorithmSource
	switch {
	case source.Provider != nil:
		// Create the config from a named algorithm provider.
		sc, err := configurator.createFromProvider(*source.Provider)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
		}
		sched = sc
	case source.Policy != nil:
		// Create the config from a user specified policy source.
		policy := &schedulerapi.Policy{}
		switch {
		case source.Policy.File != nil:
			if err := initPolicyFromFile(source.Policy.File.Path, policy); err != nil {
				return nil, err
			}
		case source.Policy.ConfigMap != nil:
			if err := initPolicyFromConfigMap(client, source.Policy.ConfigMap, policy); err != nil {
				return nil, err
			}
		}
		// Set extenders on the configurator now that we've decoded the policy
		// In this case, c.extenders should be nil since we're using a policy (and therefore not componentconfig,
		// which would have set extenders in the above instantiation of Configurator from CC options)
		configurator.extenders = policy.Extenders
		sc, err := configurator.createFromConfig(*policy)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler from policy: %v", err)
		}
		sched = sc
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
	// Additional tweaks to the config produced by the configurator.
	sched.StopEverything = stopEverything
	sched.client = client

	addAllEventHandlers(sched, informerFactory)
	return sched, nil
}
```
```
// 此示例中对no-scoring-scheduler 禁止了所有打分插件 
// pod使用.spec.schedulerName对应这里的名称  
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: no-scoring-scheduler
    plugins:
      preScore:
        disabled:
        - name: '*'
      score:
        disabled:
        - name: '*'

```
```
// 默认路径调用createFromProvider 注入profile中未更改的默认Plugins配置 并创建
// Scheduling Framework
// createFromProvider creates a scheduler from the name of a registered algorithm provider.
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}
// 配置合并
	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		prof.Plugins = plugins
	}
// create()创建Scheduler对象
	return c.create()
}
```
```
// 当实现 scheduler extender时 需要外部指定一个kind为Policy的文件 并以文件或者configmap的
// 形式注入KubeSchedulerConfiguration配置中 并在启动scheduler以 --config指定
// 注入方式 
#apiVersion: kubescheduler.config.k8s.io/v1alpha1
#kind: KubeSchedulerConfiguration
#schedulerName: my-scheduler
#algorithmSource:
#  policy:
#    file:
#      path: policy.yaml
// createFromConfig creates a scheduler from the configuration file
// Only reachable when using v1alpha1 component config
func (c *Configurator) createFromConfig(policy schedulerapi.Policy) (*Scheduler, error) {
// 准备好实际的生产方法
	lr := frameworkplugins.NewLegacyRegistry()
	args := &frameworkplugins.ConfigProducerArgs{}

	klog.V(2).Infof("Creating scheduler from configuration: %v", policy)

	// validate the policy configuration
	if err := validation.ValidatePolicy(policy); err != nil {
		return nil, err
	}

	predicateKeys := sets.NewString()
	// predicates 项为空 则取默认 否则将扩展插入
	if policy.Predicates == nil {
		klog.V(2).Infof("Using predicates from algorithm provider '%v'", schedulerapi.SchedulerDefaultProviderName)
		predicateKeys = lr.DefaultPredicates
	} else {
		for _, predicate := range policy.Predicates {
			klog.V(2).Infof("Registering predicate: %s", predicate.Name)
			predicateKeys.Insert(lr.ProcessPredicatePolicy(predicate, args))
		}
	}
  // Priorities 项为空 则取默认 负责将扩展插入
	priorityKeys := make(map[string]int64)
	if policy.Priorities 项为空 否 == nil {
		klog.V(2).Infof("Using default priorities")
		priorityKeys = lr.DefaultPriorities
	} else {
		for _, priority := range policy.Priorities {
			if priority.Name == frameworkplugins.EqualPriority {
				klog.V(2).Infof("Skip registering priority: %s", priority.Name)
				continue
			}
			klog.V(2).Infof("Registering priority: %s", priority.Name)
			priorityKeys[lr.ProcessPriorityPolicy(priority, args)] = priority.Weight
		}
	}
  // HardPodAffinityWeight决定新pod与之前存在pod在计算硬亲和时的权重 默认为1
	// HardPodAffinitySymmetricWeight in the policy config takes precedence over
	// CLI configuration.
	if policy.HardPodAffinitySymmetricWeight != 0 {
		args.InterPodAffinityArgs = &schedulerapi.InterPodAffinityArgs{
			HardPodAffinityWeight: policy.HardPodAffinitySymmetricWeight,
		}
	}

	// When AlwaysCheckAllPredicates is set to true, scheduler checks all the configured
	// predicates even after one or more of them fails.
	if policy.AlwaysCheckAllPredicates {
		c.alwaysCheckAllPredicates = policy.AlwaysCheckAllPredicates
	}

	klog.V(2).Infof("Creating scheduler with fit predicates '%v' and priority functions '%v'", predicateKeys, priorityKeys)

	// Combine all framework configurations. If this results in any duplication, framework
	// instantiation should fail.

  // 因为在Policy中没有队列排序、抢占、绑定相关的配置，这些都用默认的插件
	// "PrioritySort", "DefaultPreemption" and "DefaultBinder" were neither predicates nor priorities
	// before. We add them by default.
	plugins := schedulerapi.Plugins{
		QueueSort: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: queuesort.Name}},
		},
		PostFilter: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: defaultpreemption.Name}},
		},
		Bind: &schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{{Name: defaultbinder.Name}},
		},
	}
	var pluginConfig []schedulerapi.PluginConfig
	var err error
	// 基于插件 追加插件sets 以在policy中定义policy.predicates[0].name:PodFitsHostPorts为例
	// 其追加的插件sets为
#	registry.registerPredicateConfigProducer(PodFitsHostPortsPred,
#		func(_ ConfigProducerArgs, plugins *config.Plugins, _ *[]config.PluginConfig) {
#			plugins.Filter = appendToPluginSet(plugins.Filter, nodeports.Name, nil)
#			plugins.PreFilter = appendToPluginSet(plugins.PreFilter, nodeports.Name, nil)
#		})
		
	if plugins, pluginConfig, err = lr.AppendPredicateConfigs(predicateKeys, args, plugins, pluginConfig); err != nil {
		return nil, err
	}
	if plugins, pluginConfig, err = lr.AppendPriorityConfigs(priorityKeys, args, plugins, pluginConfig); err != nil {
		return nil, err
	}
	if pluginConfig, err = dedupPluginConfigs(pluginConfig); err != nil {
		return nil, err
	}
	for i := range c.profiles {
		prof := &c.profiles[i]
		// Plugins and PluginConfig are empty when using Policy; overriding.
		prof.Plugins = &schedulerapi.Plugins{}
		prof.Plugins.Append(&plugins)
		prof.PluginConfig = pluginConfig
	}
// create()创建Scheduler对象
	return c.create()
}

```

```
// create a scheduler from a set of registered plugins.
func (c *Configurator) create() (*Scheduler, error) {
	var extenders []framework.Extender
	var ignoredExtendedResources []string
	//c.extenders 实际就是policy中配置的extenders
	if len(c.extenders) != 0 {
		var ignorableExtenders []framework.Extender
		for ii := range c.extenders {
			klog.V(2).Infof("Creating extender with config %+v", c.extenders[ii])
			// 创建HTTPExtender对象
			extender, err := core.NewHTTPExtender(&c.extenders[ii])
			if err != nil {
				return nil, err
			}
			// 对象是否可忽略 分别进入不同的切片 主要时当一个扩展不可达或者给出错误
			// 是否scheduler应该忽略此错误 而不是失败 
			if !extender.IsIgnorable() {
				extenders = append(extenders, extender)
			} else {
				ignorableExtenders = append(ignorableExtenders, extender)
			}
			// 遍历Extender管理的资源，统计pod中Scheduler可以忽略的资源 在实现集群级别的资源管理时需要
			// 如果某 Pod 请求了此列表中的至少一个扩展资源，则 Pod 会在 filter、 prioritize 和 
			// bind   （如果扩展模块可以执行绑定操作）阶段被发送到该扩展模块
		  示例policy: 
#		  {
#  "kind": "Policy",
#  "apiVersion": "v1",
#  "extenders": [
#    {
#      "urlPrefix":"<extender-endpoint>",
#      "bindVerb": "bind",
#      "managedResources": [
#        {
#          "name": "example.com/foo",
#          "ignoredByScheduler": true
#        }
#      ]
#    }
#  ]
# }
    示例pod:
# apiVersion: v1
# kind: Pod
# metadata:
#   name: my-pod
# spec:
#   containers:
#   - name: my-container
#     image: myimage
#     resources:
#       requests:
#         cpu: 2
#         example.com/foo: 1
#       limits:
#         example.com/foo: 1


			
			for _, r := range c.extenders[ii].ManagedResources {
				if r.IgnoredByScheduler {
					ignoredExtendedResources = append(ignoredExtendedResources, r.Name)
				}
			}
		}
		// place ignorable extenders to the tail of extenders 
		extenders = append(extenders, ignorableExtenders...)
	}

	// If there are any extended resources found from the Extenders, append them to the pluginConfig for each profile.
	// This should only have an effect on ComponentConfig v1beta1, where it is possible to configure Extenders and
	// plugin args (and in which case the extender ignored resources take precedence).
	// For earlier versions, using both policy and custom plugin config is disallowed, so this should be the only
	// plugin config for this plugin.
	// 当设置 managedResources的ignoredByScheduler时 需要对每个profile中的NodeResourcesFit
	// 进行忽略此资源的断言检查
	if len(ignoredExtendedResources) > 0 {
		for i := range c.profiles {
			prof := &c.profiles[i]
			pc := schedulerapi.PluginConfig{
				Name: noderesources.FitName,
				Args: &schedulerapi.NodeResourcesFitArgs{
					IgnoredResources: ignoredExtendedResources,
				},
			}
			prof.PluginConfig = append(prof.PluginConfig, pc)
		}
	}
  // 使用同一个pod.Lister对象构造profiles即map[schedeler-name]framework.Framework对象
	// The nominator will be passed all the way to framework instantiation.
	nominator := internalqueue.NewSafePodNominator(c.informerFactory.Core().V1().Pods().Lister())
	profiles, err := profile.NewMap(c.profiles, c.registry, c.recorderFactory,
		frameworkruntime.WithClientSet(c.client),
		frameworkruntime.WithInformerFactory(c.informerFactory),
		frameworkruntime.WithSnapshotSharedLister(c.nodeInfoSnapshot),
		frameworkruntime.WithRunAllFilters(c.alwaysCheckAllPredicates),
		frameworkruntime.WithPodNominator(nominator),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(c.frameworkCapturer)),
		frameworkruntime.WithExtenders(extenders),
	)
	if err != nil {
		return nil, fmt.Errorf("initializing profiles: %v", err)
	}
	if len(profiles) == 0 {
		return nil, errors.New("at least one profile is required")
	}
	// Profiles are required to have equivalent queue sort plugins.
	lessFn := profiles[c.profiles[0].SchedulerName].QueueSortFunc()
	podQueue := internalqueue.NewSchedulingQueue(
		lessFn,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(c.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(c.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
	)

	// Setup cache debugger.
	debugger := cachedebugger.New(
		c.informerFactory.Core().V1().Nodes().Lister(),
		c.informerFactory.Core().V1().Pods().Lister(),
		c.schedulerCache,
		podQueue,
	)
	debugger.ListenForSignal(c.StopEverything)

	algo := core.NewGenericScheduler(
		c.schedulerCache,
		c.nodeInfoSnapshot,
		extenders,
		c.percentageOfNodesToScore,
	)

	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,
		Profiles:        profiles,
		NextPod:         internalqueue.MakeNextPodFunc(podQueue),
		Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
		SchedulingQueue: podQueue,
	}, nil
}
```

kube-scheduler  scheduler  outOfTreeRegistry plugin扩展方法示例(此种方法需要更改源代码 重新
编译 但不会耦合)
```
首先意图实现queenSort为例子 此例子实现了framework.QueueSortPlugin接口 实现当
优先级一致时对pod qos排序并检出
../go/src/k8s.io/kubernetes/pkg/scheduler/pkg/qos/queue_sort.go
package qos
import (
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	corev1helpers "k8s.io/component-helpers/scheduling/corev1"
	v1qos "k8s.io/kubernetes/pkg/apis/core/v1/helper/qos"
	"k8s.io/kubernetes/pkg/scheduler/framework"
)

// Name is the name of the plugin used in the plugin registry and configurations.
const Name = "QOSSort"

// Sort is a plugin that implements QoS class based sorting.
type Sort struct{}

var _ framework.QueueSortPlugin = &Sort{}

// Name returns name of the plugin.
func (pl *Sort) Name() string {
	return Name
}

// Less is the function used by the activeQ heap algorithm to sort pods.
// It sorts pods based on their priorities. When the priorities are equal, it uses
// the Pod QoS classes to break the tie.
func (*Sort) Less(pInfo1, pInfo2 *framework.QueuedPodInfo) bool {
	p1 := corev1helpers.PodPriority(pInfo1.Pod)
	p2 := corev1helpers.PodPriority(pInfo2.Pod)
	return (p1 > p2) || (p1 == p2 && compQOS(pInfo1.Pod, pInfo2.Pod))
}

func compQOS(p1, p2 *v1.Pod) bool {
	p1QOS, p2QOS := v1qos.GetPodQOS(p1), v1qos.GetPodQOS(p2)
	if p1QOS == v1.PodQOSGuaranteed {
		return true
	}
	if p1QOS == v1.PodQOSBurstable {
		return p2QOS != v1.PodQOSGuaranteed
	}
	return p2QOS == v1.PodQOSBestEffort
}

// New initializes a new plugin and returns it.
func New(_ runtime.Object, _ framework.Handle) (framework.Plugin, error) {
	return &Sort{}, nil
}
```
在main函数中注册此自定义插件
../go/src/k8s.io/kubernetes/cmd/kube-scheduler/scheduler.go
```
package main

import (
	"k8s.io/kubernetes/pkg/scheduler/pkg/qos"
	"math/rand"
	"os"
	"time"

	"github.com/spf13/pflag"

	cliflag "k8s.io/component-base/cli/flag"
	"k8s.io/component-base/logs"
	_ "k8s.io/component-base/metrics/prometheus/clientgo"
	_ "k8s.io/component-base/metrics/prometheus/version" // for version metric registration
	"k8s.io/kubernetes/cmd/kube-scheduler/app"
)

func main() {
	rand.Seed(time.Now().UnixNano())
	command := app.NewSchedulerCommand(
		app.WithPlugin(qos.Name, qos.New),
		)

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// utilflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	// utilflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```
准备好以下镜像（需要翻墙的镜像可以通过tag变换）tag不同版本不一致 在对应版本在执行make可以看到
```
k8s.gcr.io/go-runner:v2.3.1-go1.15.15-buster.0
k8s.gcr.io/kube-cross:v1.15.15-legacy-1
k8s.gcr.io/debian-iptables：buster-v1.6.5
```
执行编译镜像
```
KUBE_BASE_IMAGE_REGISTRY=k8s.gcr.io GOOS=linux GOARCH=amd64 KUBE_BUILD_PLATFORMS=linux/amd64 KUBE_BUILD_CONFORMANCE=n KUBE_BUILD_HYPERKUBE=n make release-images GOFLAGS=-v GOGCFLAGS="-N -l" KUBE_BUILD_PULL_LATEST_IMAGES=false
镜像tar包位于 ../go/src/k8s.io/kubernetes/_output/release-images/amd64
```
配置对应插件的启用及禁用
```
// apiVersion会随着k8s主版本变化 此kind变化较大 注意版本
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
leaderElection:
  # (Optional) Change true to false if you are not running a HA control-plane.
  leaderElect: false
clientConnection:
  kubeconfig: /etc/kubernetes/kube-scheduler-kubeconfig.yaml
profiles:
- schedulerName: default-scheduler
  plugins:
    queueSort:
      enabled:
      - name: QOSSort
      disabled:
      - name: "*"
queueSort 插件本身不会使用一个列表 只会使用一个 因此这里需要禁止原本的插件
scheduler启动成功 即ok 打开v==2日志 可以看到上文提到的配置列表 观察插件是否启用
```
##### 1.queuesort
- 关键代码
```
// Less is the function used by the activeQ heap algorithm to sort pods.
// It sorts pods based on their priority. When priorities are equal, it uses
// PodQueueInfo.timestamp.
func (pl *PrioritySort) Less(pInfo1, pInfo2 *framework.QueuedPodInfo) bool {
	p1 := corev1helpers.PodPriority(pInfo1.Pod)
	p2 := corev1helpers.PodPriority(pInfo2.Pod)
	return (p1 > p2) || (p1 == p2 && pInfo1.Timestamp.Before(pInfo2.Timestamp))
}
```
默认采取此queuesort,在队列中的pod按照优先级检出

##### 2.PreFilter
- noderesources
```
func (f *Fit) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
	cycleState.Write(preFilterStateKey, computePodResourceRequest(pod))
	return nil
}

func computePodResourceRequest(pod *v1.Pod) *preFilterState {
	result := &preFilterState{}
	for _, container := range pod.Spec.Containers {
		result.Add(container.Resources.Requests)
	}

	// take max_resource(sum_pod, any_init_container)
	for _, container := range pod.Spec.InitContainers {
		result.SetMaxResource(container.Resources.Requests)
	}

	// If Overhead is being utilized, add to the total requests for the pod
	if pod.Spec.Overhead != nil && utilfeature.DefaultFeatureGate.Enabled(features.PodOverhead) {
		result.Add(pod.Spec.Overhead)
	}

	return result
}
```
计算pod实例中的资源request总值
result=max(普通容器request加和,init容器资源最大)+Overhead

- nodeports
```
func (pl *NodePorts) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
	s := getContainerPorts(pod)
	cycleState.Write(preFilterStateKey, preFilterState(s))
	return nil
}

func getContainerPorts(pods ...*v1.Pod) []*v1.ContainerPort {
	ports := []*v1.ContainerPort{}
	for _, pod := range pods {
		for j := range pod.Spec.Containers {
			container := &pod.Spec.Containers[j]
			for k := range container.Ports {
				ports = append(ports, &container.Ports[k])
			}
		}
	}
	return ports
}
```
ports计算方法  遍历pod中的容器端口 以port号:true 写入结果

- podtopologyspread
```
func (pl *PodTopologySpread) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
	s, err := pl.calPreFilterState(pod)
	if err != nil {
		return framework.AsStatus(err)
	}
	cycleState.Write(preFilterStateKey, s)
	return nil
}

func (pl *PodTopologySpread) calPreFilterState(pod *v1.Pod) (*preFilterState, error) {
	allNodes, err := pl.sharedLister.NodeInfos().List()
	if err != nil {
		return nil, fmt.Errorf("listing NodeInfos: %v", err)
	}
	var constraints []topologySpreadConstraint
	//获取pod设置的pod间的拓扑分布约束配置 将硬策略添加到constraints列表中
	//存储形式为topologySpreadConstraint对象切片
	//result = append(result, topologySpreadConstraint{
	//			MaxSkew:     c.MaxSkew,
	//			TopologyKey: c.TopologyKey,
	//			Selector:    selector,
	//		})
	//否则使用默认的pod间拓扑分布约束
	//var systemDefaultConstraints = []v1.TopologySpreadConstraint{
	//{
	//	TopologyKey:       v1.LabelHostname,
	//	WhenUnsatisfiable: v1.ScheduleAnyway,
	//	MaxSkew:           3,
	//},
	//{
	//	TopologyKey:       v1.LabelTopologyZone,
	//	WhenUnsatisfiable: v1.ScheduleAnyway,
	//	MaxSkew:           5,
	//},
  //}默认情况下都属软策略
			
	if len(pod.Spec.TopologySpreadConstraints) > 0 {
		// We have feature gating in APIServer to strip the spec
		// so don't need to re-check feature gate, just check length of Constraints.
		constraints, err = filterTopologySpreadConstraints(pod.Spec.TopologySpreadConstraints, v1.DoNotSchedule)
		if err != nil {
			return nil, fmt.Errorf("obtaining pod's hard topology spread constraints: %v", err)
		}
	} else {
		constraints, err = pl.buildDefaultConstraints(pod, v1.DoNotSchedule)
		if err != nil {
			return nil, fmt.Errorf("setting default hard topology spread constraints: %v", err)
		}
	}
	if len(constraints) == 0 {
		return &preFilterState{}, nil
	}

	s := preFilterState{
		Constraints:          constraints,
		TpKeyToCriticalPaths: make(map[string]*criticalPaths, len(constraints)),
		TpPairToMatchNum:     make(map[topologyPair]*int32, sizeHeuristic(len(allNodes), constraints)),
	}
	for _, n := range allNodes {
		node := n.Node()
		if node == nil {
			klog.Error("node not found")
			continue
		}
		// In accordance to design, if NodeAffinity or NodeSelector和 is defined,
		// spreading is applied to nodes that pass those filters.
		// 当node不适合NodeSelector和NodeAffinity 直接跳过
		if !helper.PodMatchesNodeSelectorAndAffinityTerms(pod, node) {
			continue
		}
		// Ensure current node's labels contains all topologyKeys in 'Constraints'.
		// 确认当前node 在对应的拓扑域规则中
		if !nodeLabelsMatchSpreadConstraints(node.Labels, constraints) {
			continue
		}
		//初始化上文的TpPairToMatchNum结构 以节点拓扑域作为key 值为此域中对应ns下
		//匹配的pod数量
		for _, c := range constraints {
			pair := topologyPair{key: c.TopologyKey, value: node.Labels[c.TopologyKey]}
			s.TpPairToMatchNum[pair] = new(int32)
		}
	}

	processNode := func(i int) {
		nodeInfo := allNodes[i]
		node := nodeInfo.Node()

		for _, constraint := range constraints {
			pair := topologyPair{key: constraint.TopologyKey, value: node.Labels[constraint.TopologyKey]}
			tpCount := s.TpPairToMatchNum[pair]
			if tpCount == nil {
				continue
			}
			// 1.注意这里是排除终止态pod的 是否排除终止态pod是由每个过滤器自己决定(调度器同时处理
			// assignedPod和unassignedPod)
			// 2.返回在此Node对应空间中匹配标签pod的数量
			count := countPodsMatchSelector(nodeInfo.Pods, constraint.Selector, pod.Namespace)
			atomic.AddInt32(tpCount, int32(count))
		}
	}
	//计算所有node
	parallelize.Until(context.Background(), len(allNodes), processNode)

	// calculate min match for each topology pair
	//计算每一个拓扑域的最小匹配 把所有约束中匹配的 Pod 数量最小（或稍大）的放入 TpKeyToCriticalPaths 为了均匀 一般我们只会取最小匹配的区域
	for i := 0; i < len(constraints); i++ {
		key := constraints[i].TopologyKey
		s.TpKeyToCriticalPaths[key] = newCriticalPaths()
	}
	for pair, num := range s.TpPairToMatchNum {
		s.TpKeyToCriticalPaths[pair.key].update(pair.value, *num)
	}

	return &s, nil
}
```

这里主要是对节点和pod间的拓扑分布约束进行计算 在pod的配置中体现如下 此特性需要开启EvenPodsSpread特性门  (>v1.16,default by 1.18)
```
spec:
  topologySpreadConstraints:
  - maxSkew: <integer>
    topologyKey: <string>
    whenUnsatisfiable: <string>
    labelSelector: <object>
```

首先 
labelSelector决定用来查找匹配的 Pod,对每个拓扑域中的pod数量就行统计
topologyKey决定节点拓扑域的划分,具有同样的key,同样label值的节点将被抽象化为一个拓扑域
maxSkew描述了 Pod 在不同拓扑域中不均匀分布的最大程度，maxSkew 的取值必须大于 0。
每个拓扑域都有一  个 skew，计算的公式是: skew[i] = 拓扑域[i]中匹配的 Pod 个数 - min{其他拓扑域中匹配的 Pod 
个数}
whenUnsatisfiable当不满足分布约束条件该采取何种策略 DoNotSchedule(硬策略)  ScheduleAnyway(依据skew排序Node调度)

这里只准备了min{其他拓扑域中匹配的 Pod 个数}等数据  具体需要在filter中分析
- interpodaffinity
```
// PreFilter invoked at the prefilter extension point.
func (pl *InterPodAffinity) PreFilter(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod) *framework.Status {
	var allNodes []*framework.NodeInfo
	var nodesWithRequiredAntiAffinityPods []*framework.NodeInfo
	var err error
	if allNodes, err = pl.sharedLister.NodeInfos().List(); err != nil {
		return framework.NewStatus(framework.Error, fmt.Sprintf("failed to list NodeInfos: %v", err))
	}
	if nodesWithRequiredAntiAffinityPods, err = pl.sharedLister.NodeInfos().HavePodsWithRequiredAntiAffinityList(); err != nil {
		return framework.NewStatus(framework.Error, fmt.Sprintf("failed to list NodeInfos with pods with affinity: %v", err))
	}

	podInfo := framework.NewPodInfo(pod)
	if podInfo.ParseError != nil {
		return framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("parsing pod: %+v", podInfo.ParseError))
	}
	// existingPodAntiAffinityMap will be used later for efficient check on existing pods' anti-affinity
	existingPodAntiAffinityMap := getTPMapMatchingExistingAntiAffinity(pod, nodesWithRequiredAntiAffinityPods)

	// incomingPodAffinityMap will be used later for efficient check on incoming pod's affinity
	// incomingPodAntiAffinityMap will be used later for efficient check on incoming pod's anti-affinity
	incomingPodAffinityMap, incomingPodAntiAffinityMap := getTPMapMatchingIncomingAffinityAntiAffinity(podInfo, allNodes)

	s := &preFilterState{
		topologyToMatchedAffinityTerms:             incomingPodAffinityMap,
		topologyToMatchedAntiAffinityTerms:         incomingPodAntiAffinityMap,
		topologyToMatchedExistingAntiAffinityTerms: existingPodAntiAffinityMap,
		podInfo: podInfo,
	}

	cycleState.Write(preFilterStateKey, s)
	return nil
}
```










