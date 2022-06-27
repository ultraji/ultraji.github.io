---
title: Kubernetes组件学习 - kube-controller-manager
categories:
  - kubernetes
permalink: /:categories/how-kube-controller-manager-work
tags:
  - kubernetes
  - 云原生之路
---

kube-controller-manager如字面意思一样，是**控制器的管理者**，用于管理一系列控制器，例如Node Controller、Job Controller、Deployment Controller、Endpoints Controller、Namespace Controller等。这些控制器的职责是保证集群中各种资源的状态和用户定义（yaml）的状态一致，如果出现偏差，则进行修正。

<!--more-->

![kuber-controller-manager](/assets/images/cloud-native/2022-03-21-kube-controller-manager-01.png){:height="75%" width="75%"}

可在 `cmd/kube-controller-manager/app/controllermanager.go` 的`NewControllerInitializers`函数中找到详细的controller列表。

```go
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["endpointslice"] = startEndpointSliceController
	...
	controllers["ephemeral-volume"] = startEphemeralVolumeController
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) {
		controllers["storage-version-gc"] = startStorageVersionGCController
	}

	return controllers
}
```

## 启动流程

controller-manager启动的主要逻辑都在`cmd/kube-controller-manager/app/controllermanager.go`文件中。

controller-manager是通过二进制加命令行参数的方式（通过cobra库实现的，即`NewControllerManagerCommand`）启动运行的。

controller-manager的核心运行逻辑由`Run`函数实现。在kubernetes中，对controller-manager而言，可能同时运行着多个副本。由**选主逻辑**保证只有一个副本会运行业务逻辑，其他副本则处于假死状态。（选主逻辑：即下面的`leaderElectAndRun`方法，`OnStartedLeading`中的内容为获得锁，处于leader地位所运行的逻辑。）

```go
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
    ...

	// Start the main lock
	go leaderElectAndRun(c, id, electionChecker,
		c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		c.ComponentConfig.Generic.LeaderElection.ResourceName,
		leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				initializersFunc := NewControllerInitializers
				if leaderMigrator != nil {
					// If leader migration is enabled, we should start only non-migrated controllers
					//  for the main lock.
					initializersFunc = createInitializersFunc(leaderMigrator.FilterFunc, leadermigration.ControllerNonMigrated)
					klog.Info("leader migration: starting main controllers.")
				}
				run(ctx, startSATokenController, initializersFunc)
			},
			OnStoppedLeading: func() {
				klog.ErrorS(nil, "leaderelection lost")
				klog.FlushAndExit(klog.ExitFlushTimeout, 1)
			},
		})

	...
}
```

`run(ctx, startSATokenController, initializersFunc)`可知，**`controller-manager`启动后的唯一动作就是运行起所有controllers**。这里注意到，`startSATokenController`从众多controllers中被单独拎出来，作为初始化的controller第一个运行，可以从`run`函数中的`StartControllers`的逻辑中可以找到答案（即为其他controller提供凭证）。

```go
func StartControllers(ctx context.Context, controllerCtx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc,
	unsecuredMux *mux.PathRecorderMux, healthzHandler *controllerhealthz.MutableHealthzHandler) error {
	// Always start the SA token controller first using a full-power client, since it needs to mint tokens for the rest
	// If this fails, just return here and fail since other controllers won't be able to get credentials.
	if startSATokenController != nil {
		if _, _, err := startSATokenController(ctx, controllerCtx); err != nil {
			return err
		}
	}
	...
}
```

至此，controller-manager启动完成，其所做也只是简单地运行起所有controller，然后由每个controller实现自身的业务逻辑。

## 核心逻辑

接下来，具体的各种controller的逻辑实现则在`pkg/controller`文件夹下。