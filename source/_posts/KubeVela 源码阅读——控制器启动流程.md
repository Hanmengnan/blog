---
title: KubeVela 源码阅读——控制器启动
author: siegelion
date: 2022/7/10 15:15
tags: [KubeVela, 云原生]
categories: [笔记]
---



# KubeVela

KubeVela以应用为中心，通过OAM（开放应用模型）作为核心API进行构建，同时KubeVela也是OAM的最佳实践。

KubeVela中将应用拆分为各种细粒度的组件，并通过CRD注册到K8S中。同时KubeVela通过引入CUE作为模板引擎，提供了对CRD的抽象，用户可以用过编写CUE定义的应用模板，将多种K8S资源组装在一起构成应用。

# 控制器启动

我们从`cmd/core/main.go`文件的`main`函数入手，这里是核心资源以及控制器启动的入口。控制器启动是一个漫长的过程，有很多步骤，下图是控制器启动的流程示意图。

![控制器启动流程](https://siegelion-blog.oss-cn-beijing.aliyuncs.com/blog/image-20220713083026298.png)

## 绑定变量

从命令行读入参数或采用默认值绑定至变量

```go
// 是否开启 Admission Webhook
flag.BoolVar(&useWebhook, "use-webhook", false, "Enable Admission Webhook")
// 	Admission webhook 证书目录
flag.StringVar(&certDir, "webhook-cert-dir", "/k8s-webhook-server/serving-certs", "Admission webhook cert/key dir.")
// Admission webhook 监听端口
flag.IntVar(&webhookPort, "webhook-port", 9443, "admission webhook listen address")
// Prometheus exporter 监听地址
flag.StringVar(&metricsAddr, "metrics-addr", ":8080", "The address the metric endpoint binds to.")
// 是否开启控制器主阶段选举。开启之后同一时间只允许一个控制器处于激活状态， 其他副本控制器将处于休眠状态。
flag.BoolVar(&enableLeaderElection, "enable-leader-election", false,
             "Enable leader election for controller manager. Enabling this will ensure there is only one active controller manager.")
// 定义主节点选举 configmap 所在 namespace 默认处于 default namespace 下
flag.StringVar(&leaderElectionNamespace, "leader-election-namespace", "",
               "Determines the namespace in which the leader election configmap will be created.")
// 日志落盘目录
flag.StringVar(&logFilePath, "log-file-path", "", "The file to write logs to.")
// 日志最大体积
flag.Uint64Var(&logFileMaxSize, "log-file-max-size", 1024, "Defines the maximum size a log file can grow to, Unit is megabytes.")
// 是否开启debug日志
flag.BoolVar(&logDebug, "log-debug", false, "Enable debug logs for development purpose")
flag.IntVar(&controllerArgs.RevisionLimit, "revision-limit", 50,
            "RevisionLimit is the maximum number of revisions that will be maintained. The default value is 50.")
// 没有使用应用的版本，可能用于垃圾回收，下同
flag.IntVar(&controllerArgs.AppRevisionLimit, "application-revision-limit", 10,
            "application-revision-limit is the maximum number of application useless revisions that will be maintained, if the useless revisions exceed this number, older ones will be GCed first.The default value is 10.")
flag.IntVar(&controllerArgs.DefRevisionLimit, "definition-revision-limit", 20,
            "definition-revision-limit is the maximum number of component/trait definition useless revisions that will be maintained, if the useless revisions exceed this number, older ones will be GCed first.The default value is 20.")
flag.StringVar(&controllerArgs.CustomRevisionHookURL, "custom-revision-hook-url", "",
               "custom-revision-hook-url is a webhook url which will let KubeVela core to call with applicationConfiguration and component info and return a customized component revision")
flag.BoolVar(&controllerArgs.AutoGenWorkloadDefinition, "autogen-workload-definition", true, "Automatic generated workloadDefinition which componentDefinition refers to.")
// 健康检查接口监听地址
flag.StringVar(&healthAddr, "health-addr", ":9440", "The address the health endpoint binds to.")
// 如果spec没有发生变化，不允许进行apply操作
flag.StringVar(&applyOnceOnly, "apply-once-only", "false",
               "For the purpose of some production environment that workload or trait should not be affected if no spec change, available options: on, off, force.")
// 	是否禁用的内置功能列表
flag.StringVar(&disableCaps, "disable-caps", "", "To be disabled builtin capability list.")
// 	Application 文件存放的存储介质，默认为本地存放
flag.StringVar(&storageDriver, "storage-driver", "Local", "Application file save to the storage driver")
flag.DurationVar(&commonconfig.ApplicationReSyncPeriod, "application-re-sync-period", 5*time.Minute,
                 "Re-sync period for application to re-sync, also known as the state-keep interval.")
flag.DurationVar(&commonconfig.ReconcileTimeout, "reconcile-timeout", time.Minute*3,
                 "the timeout for controller reconcile")
...
```

## 配置日志设置

```go
if logDebug {
    _ = flag.Set("v", strconv.Itoa(int(commonconfig.LogDebug)))
}
...
if logFilePath != "" {
    _ = flag.Set("logtostderr", "false")
    _ = flag.Set("log_file", logFilePath)
    _ = flag.Set("log_file_max_size", strconv.FormatUint(logFileMaxSize, 10))
}
```



## pprof 服务启动

```go
// pprof server setup
if pprofAddr != "" {
    // Start pprof server if enabled
    mux := http.NewServeMux()
    mux.HandleFunc("/debug/pprof/", pprof.Index)
    mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
    pprofServer := http.Server{
        Addr:    pprofAddr,
        Handler: mux,
    }
    klog.InfoS("Starting debug HTTP server", "addr", pprofServer.Addr)

    go func() {
        go func() {
            ctx := context.Background()
            <-ctx.Done()

            ctx, cancelFunc := context.WithTimeout(context.Background(), 60*time.Minute)
            defer cancelFunc()

            if err := pprofServer.Shutdown(ctx); err != nil {
                klog.Error(err, "Failed to shutdown debug HTTP server")
            }
        }()

        if err := pprofServer.ListenAndServe(); !errors.Is(http.ErrServerClosed, err) {
            klog.Error(err, "Failed to start debug HTTP server")
            panic(err)
        }
    }()
}
```

## 加载restConfig

各个控制器作为API-Server的客户端，这里加载的配置即为API-Server与客户端交互的配置，这些配置会影响客户端与API-Server之间的交互。

这里对`UserAgent`设置的目的是为了方便在日志中排查是哪个 pod 对 API-server 发起了请求。对 QPS 和 Burst 的设置限制了 API-Server 从客户端收到请求的每秒最高并发数，通过限制从客户端处进行限流保证 API-Server 的处于一个可用的状态，不会因为某些控制器的高频请求而导致整个集群的宕机。

```go
restConfig := ctrl.GetConfigOrDie()
restConfig.UserAgent = types.KubeVelaName + "/" + version.GitRevision
restConfig.QPS = float32(qps)
restConfig.Burst = burst
restConfig.Wrap(auth.NewImpersonatingRoundTripper)
```

## 多集群环境配置

```go
// wrapper the round tripper by multi cluster rewriter
if enableClusterGateway {
    client, err := multicluster.Initialize(restConfig, true)
    if err != nil {
        klog.ErrorS(err, "failed to enable multi-cluster capability")
        os.Exit(1)
    }

    if enableClusterMetrics {
        _, err := multicluster.NewClusterMetricsMgr(context.Background(), client, clusterMetricsInterval)
        if err != nil {
            klog.ErrorS(err, "failed to enable multi-cluster-metrics capability")
            os.Exit(1)
        }
    }
}
```

## 控制器管理器初始化

```go
mgr, err := ctrl.NewManager(restConfig, ctrl.Options{
    Scheme:                     scheme,
    MetricsBindAddress:         metricsAddr,
    LeaderElection:             enableLeaderElection,
    LeaderElectionNamespace:    leaderElectionNamespace,
    LeaderElectionID:           leaderElectionID,
    Port:                       webhookPort,
    CertDir:                    certDir,
    HealthProbeBindAddress:     healthAddr,
    LeaderElectionResourceLock: leaderElectionResourceLock,
    LeaseDuration:              &leaseDuration,
    RenewDeadline:              &renewDeadline,
    RetryPeriod:                &retryPeriod,
    // SyncPeriod is configured with default value, aka. 10h. First, controller-runtime does not
    // recommend use it as a time trigger, instead, it is expected to work for failure tolerance
    // of controller-runtime. Additionally, set this value will affect not only application
    // controller but also all other controllers like definition controller. Therefore, for
    // functionalities like state-keep, they should be invented in other ways.
    NewClient: ctrlClient.DefaultNewControllerClient,
})
```

## 注册健康检查

```go
if err := registerHealthChecks(mgr); err != nil {
    klog.ErrorS(err, "Unable to register ready/health checks")
    os.Exit(1)
}
```

## 检查被禁用控制器是否合法

```go
if err := utils.CheckDisabledCapabilities(disableCaps); err != nil {
    klog.ErrorS(err, "Unable to get enabled capabilities")
    os.Exit(1)
}
```

## 加载控制器应用模式

控制器存在三种应用模式：`applyOnceOnly`、`ApplyOnceOnlyOn`、`ApplyOnceOnlyForce`。

| 状态               | 解释                                                         |
| ------------------ | ------------------------------------------------------------ |
| ApplyOnceOnlyOff   | 关闭 ApplyOnceOnlyOn                                         |
| ApplyOnceOnlyOn    | workload 或 trait 只会应用一次，在没有明确指明对其本身进行修改的情况下，修改其他组件进而影响该组件状态变化时，workload 或 trait 并不会重新应用 |
| ApplyOnceOnlyForce | 不受其他组件的影响的程度比 ApplyOnceOnlyOn 更加顽固，即使收到其他组件的影响，导致 workload 或 trait 被删除的情况下，依然不会重新应用 |



```go
switch strings.ToLower(applyOnceOnly) {
	case "", "false", string(oamcontroller.ApplyOnceOnlyOff):
		controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyOff
		klog.Info("ApplyOnceOnly is disabled")
	case "true", string(oamcontroller.ApplyOnceOnlyOn):
		controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyOn
		klog.Info("ApplyOnceOnly is enabled, that means workload or trait only apply once if no spec change even they are changed by others")
	case string(oamcontroller.ApplyOnceOnlyForce):
		controllerArgs.ApplyMode = oamcontroller.ApplyOnceOnlyForce
		klog.Info("ApplyOnceOnlyForce is enabled, that means workload or trait only apply once if no spec change even they are changed or deleted by others")
	default:
		klog.ErrorS(fmt.Errorf("invalid apply-once-only value: %s", applyOnceOnly),
			"Unable to setup the vela core controller",
			"apply-once-only", "on/off/force, by default it's off")
		os.Exit(1)
	}
```

## 注册资源发现客户端

### K8S资源

这个函数返回一个`DiscoveryMapper`，根据他的注释可以将其看作一个从GVK获取资源的客户端。

> DiscoveryMapper is a interface for refresh and discovery resources from GVK.

```go
dm, err := discoverymapper.New(mgr.GetConfig())
```

### CUE资源

这个函数注册了从K8S集群中获取CUE包的客户端。

> PackageDiscover defines the inner CUE packages loaded from K8s cluster

```go
pd, err := packages.NewPackageDiscover(mgr.GetConfig())
```

## 注册Webhook

```go
if useWebhook {
    klog.InfoS("Enable webhook", "server port", strconv.Itoa(webhookPort))
    oamwebhook.Register(mgr, controllerArgs)
    if err := waitWebhookSecretVolume(certDir, waitSecretTimeout, waitSecretInterval); err != nil {
        klog.ErrorS(err, "Unable to get webhook secret")
        os.Exit(1)
    }
}
```

## 注册控制器集

### OAM控制器集

内部包含很多核心控制器，包括`application`、`trait`、`component`、`policy`、`workflow`多个控制器。

```go
if err = oamv1alpha2.Setup(mgr, controllerArgs); err != nil {
    klog.ErrorS(err, "Unable to setup the oam controller")
    os.Exit(1)
}
```

### Vela控制器集

```go
if err = standardcontroller.Setup(mgr, disableCaps, controllerArgs); err != nil {
    klog.ErrorS(err, "Unable to setup the vela core controller")
    os.Exit(1)
}
```

## 启动应用指标检测

```go
informer, err := mgr.GetCache().GetInformer(context.Background(), &v1beta1.Application{})
if err != nil {
    klog.ErrorS(err, "Unable to get informer for application")
}
watcher.StartApplicationMetricsWatcher(informer)
```

## 控制器管理器启动

```go
if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
    klog.ErrorS(err, "Failed to run manager")
    os.Exit(1)
}
```

