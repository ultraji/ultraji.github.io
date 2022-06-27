---
title: Kubernetes组件高可用的实现 - leaderelection选举策略
categories:
  - kubernetes
permalink: /:categories/k8s-leaderelection-implementation
tags: 
  - kubernetes
  - 云原生之路
---

在kubernetes中，同一个组件会存在很多副本。对于apiserver而言，每个副本都会运行业务逻辑，对于scheduler和controller-manager而言，同时只有一个副本会运行业务逻辑，其他副本则处于假死状态。而这种确保只有一个副本处于业务逻辑的功能是由kubernetes中的leaderelection实现的。

<!--more-->

## 概述

简单来说，所有运行中的副本都会去抢占一个资源锁，抢到锁的副本就获得了执行业务逻辑的权力。kubernetes中提供了endpointsleases、configmapsleases、leases三种资源锁类型。可以通过`kubectl get leases -nkube-system`看到锁信息。

```
root@master:~# kubectl get leases -nkube-system
NAME                      HOLDER                                        AGE
kube-controller-manager   master_f1048258-e944-45b8-ac5b-cb9a149ff846   233d
kube-scheduler            master_bacfd3e0-ffca-4840-96ea-312b8e2f4661   233d
```

查看具体的锁信息，可知当前所被`master_f1048258-e944-45b8-ac5b-cb9a149ff846`实例（hostname + uid组成）占用。

```
root@master:~# kubectl get leases kube-controller-manager -nkube-system -oyaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2021-07-11T14:32:20Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "17857890"
  uid: def37dbc-8db4-4ef8-a471-02e645483ea6
spec:
  acquireTime: "2021-07-24T09:11:56.098877Z"
  holderIdentity: master_f1048258-e944-45b8-ac5b-cb9a149ff846
  leaseDurationSeconds: 15
  leaseTransitions: 13
  renewTime: "2021-07-30T13:57:04.800422Z"
```


## leaderelection选举策略是如何使用的

以controller-manager为例，在`kubernetes/cmd/kube-controller-manager/app/controllermanager.go`中。

```go
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
  ...

  // Start the main lock
  go leaderElectAndRun(c, id, electionChecker,
    c.ComponentConfig.Generic.LeaderElection.ResourceLock,
    c.ComponentConfig.Generic.LeaderElection.ResourceName,
    leaderelection.LeaderCallbacks{
      // 成为leader后的执行逻辑：
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
      // 失去leader地位后的执行逻辑：
      OnStoppedLeading: func() {
        klog.Fatalf("leaderelection lost")
      },
    })

  ...
}
```

`leaderElectAndRun`实际上是对`leaderelection.RunOrDie`的简单封装。

```go
func leaderElectAndRun(c *config.CompletedConfig, lockIdentity string, electionChecker *leaderelection.HealthzAdaptor, resourceLock string, leaseName string, callbacks leaderelection.LeaderCallbacks) {
  ...

  leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
    // Lock:          资源锁对象
    // LeaseDuration: 租约时长，非leader的副本用来判断资源锁是否过期
    // RenewDeadline: leader获取资源锁的超时时间，超过这个时间将放弃资源锁
    // RetryPeriod:   重试间隔，俩次尝试获取锁之间应该等待的时间
    // Callbacks:     回调函数，由leadereletion的事件触发对应的逻辑，如上面Run代码所示
    // WatchDog:      用于健康检查，非必须
    // Name:          资源锁的名字，用于debug打印
    Lock:          rl,
    LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
    RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
    RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
    Callbacks:     callbacks,
    WatchDog:      electionChecker,
    Name:          leaseName,
  })

  panic("unreachable")
}
```


## leaderelection选举策略是如何实现的

leaderelection实现代码在`client-go/tools/leaderelection/leaderelection.go`中。leaderelection实现的主要功能是根据**提供的锁模型**和**配置的各种回调函数**保证业务逻辑的独一份运行。

```go
func (le *LeaderElector) Run(ctx context.Context) {
  defer runtime.HandleCrash()
  defer func() {
    le.config.Callbacks.OnStoppedLeading()
  }()

  // 尝试获取锁，失败即退出
  if !le.acquire(ctx) {
    return // ctx signalled done
  }
  // 获取锁后，执行OnStartedLeading回调函数，通过renew中的代码持续刷新锁状态
  ctx, cancel := context.WithCancel(ctx)
  defer cancel()
  go le.config.Callbacks.OnStartedLeading(ctx)
  le.renew(ctx)
  // renew刷新锁失败后退出循环，此时Run函数结束，执行defer中的OnStoppedLeading回调函数
}
```

在le.acquire和le.renew中调用tryAcquireOrRenew函数来获取或刷新锁。**抢占锁的动作是通过将自己的信息更新到锁上实现的。**

```go
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
  ...
  // 1. obtain or create the ElectionRecord
  // 1. 第一步，获取一个锁信息的记录，若不存在记录，则创建一个。
  oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
  if err != nil {
    if !errors.IsNotFound(err) {
      klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
      return false
    }
    if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
      klog.Errorf("error initially creating leader election record: %v", err)
      return false
    }

    le.setObservedRecord(&leaderElectionRecord)

    return true
  }

  // 2. Record obtained, check the Identity & Time
  // 2. 将刚获取的锁信息记录下来（加一层byte Equal为了减少更新记录的次数）；非leader检查锁信息后，发现锁未过期，则打印日志后退出
  if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
    le.setObservedRecord(oldLeaderElectionRecord)

    le.observedRawRecord = oldLeaderElectionRawRecord // 单纯为了比较动作快一点，这里同时会把byte流保存下来，然后通过byte流比较
  }
  if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
    le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
    !le.IsLeader() {
    klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
    return false
  }

  // 3. We're going to try to update. The leaderElectionRecord is set to it's default
  // here. Let's correct it before updating.
  // 3. 如果是leader，则尝试刷新锁；如果是非leader，则尝试抢占锁。刷新和抢占锁是通过将自己的信息更新到锁上实现的。
  if le.IsLeader() {
    leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
    leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
  } else {
    leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
  }

  // update the lock itself 
  if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
    klog.Errorf("Failed to update lock: %v", err)
    return false
  }

  le.setObservedRecord(&leaderElectionRecord)
  return true
}
```
