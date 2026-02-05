---
pubDate: 2022-01-09
title: Pod容器内的resolv.conf配置是如何生成的
tags: 
  - kubernetes
  - 云原生之路
postId: "220101"
---

在`resolv.conf`配置了容器内的DNS服务器信息，决定了容器内的域名搜索顺序。那么容器内的`resolv.conf`是如何生成的呢。

<!--more-->

* Pod的dnsPolicy策略
* kubelet如何获得集群的DNS服务地址
* resolv.conf如何生成

## Pod的dnsPolicy策略

首先需要理解dnsPolicy概念，Kubernetes中Pod的dnsPolicy策略有**4**种类型（详见[https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)）：

* **Default**：Pod从运行所在的节点（即宿主机）继承名称解析配置
* **ClusterFirst**：**k8s的默认设置**，与配置的集群域后缀匹配的查询会访问cluster的DNS服务，后缀不匹配的查询会转发到继承自主机的上游DNS服务器
* **ClusterFirstWithHostNet**：对于以hostNetwork方式运行的Pod，应将其DNS策略显式设置为 ClusterFirstWithHostNet。否则，以hostNetwork方式和ClusterFirst策略运行的Pod将会做出回退至Default策略的行为
* **None**：Pod会使用其dnsConfig字段所提供的DNS设置



### kubelet如何获得集群的DNS服务地址

kubelet在启动时可通过`--cluster-dns`参数来配置ClusterDNS , 当pod的dns策略为ClusterFirst时, 则使用kubelet的该参数。通常该参数指定为kubedns服务的地址。


### resolv.conf如何生成

首先，kubelet收到Pod创建事件后, 调用`SyncPod`来创建pod，在创建pod的第4步（即创建pause容器时）调用`createPodSandbox`方法生成对应的resolv.conf配置。

源码位置：`pkg/kubelet/kuberuntime/kuberuntime_manager.go`

```go
// SyncPod syncs the running pod into the desired pod by executing following steps:
func (m *kubeGenericRuntimeManager) SyncPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	...
   	
	// Step 4: Create a sandbox for the pod if necessary.
	podSandboxID := podContainerChanges.SandboxID
	if podContainerChanges.CreateSandbox {
		...
        // 此处创建pause容器
		podSandboxID, msg, err = m.createPodSandbox(ctx, pod, podContainerChanges.Attempt)
		...
}
```

`createPodSandbox`第一步`generatePodSandboxConfig`则会生成dns配置信息。

源码位置：`pkg/kubelet/kuberuntime/kuberuntime_sandbox.go`

```go
// createPodSandbox creates a pod sandbox and returns (podSandBoxID, message, error).
func (m *kubeGenericRuntimeManager) createPodSandbox(ctx context.Context, pod *v1.Pod, attempt uint32) (string, string, error) {
    // 
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
	...
}

// generatePodSandboxConfig generates pod sandbox config from v1.Pod.
func (m *kubeGenericRuntimeManager) generatePodSandboxConfig(pod *v1.Pod, attempt uint32) (*runtimeapi.PodSandboxConfig, error) {
	...

	dnsConfig, err := m.runtimeHelper.GetPodDNS(pod)
	if err != nil {
		return nil, err
	}
	podSandboxConfig.DnsConfig = dnsConfig
	...
    
	return podSandboxConfig, nil
}
```

源码位置：`pkg/kubelet/network/dns/dns.go`

```go
// GetPodDNS returns DNS settings for the pod.
func (c *Configurer) GetPodDNS(pod *v1.Pod) (*runtimeapi.DNSConfig, error) {
	dnsConfig, err := c.getHostDNSConfig()
	if err != nil {
		return nil, err
	}

	dnsType, err := getPodDNSType(pod)
	if err != nil {
		klog.ErrorS(err, "Failed to get DNS type for pod. Falling back to DNSClusterFirst policy.", "pod", klog.KObj(pod))
		dnsType = podDNSCluster
	}
	switch dnsType {
	case podDNSNone:
		// DNSNone should use empty DNS settings as the base.
		dnsConfig = &runtimeapi.DNSConfig{}
	case podDNSCluster:
		if len(c.clusterDNS) != 0 {
			// For a pod with DNSClusterFirst policy, the cluster DNS server is
			// the only nameserver configured for the pod. The cluster DNS server
			// itself will forward queries to other nameservers that is configured
			// to use, in case the cluster DNS server cannot resolve the DNS query
			// itself.
			dnsConfig.Servers = []string{}
			for _, ip := range c.clusterDNS {
				dnsConfig.Servers = append(dnsConfig.Servers, ip.String())
			}
			dnsConfig.Searches = c.generateSearchesForDNSClusterFirst(dnsConfig.Searches, pod)
			dnsConfig.Options = defaultDNSOptions
			break
		}
		// clusterDNS is not known. Pod with ClusterDNSFirst Policy cannot be created.
		nodeErrorMsg := fmt.Sprintf("kubelet does not have ClusterDNS IP configured and cannot create Pod using %q policy. Falling back to %q policy.", v1.DNSClusterFirst, v1.DNSDefault)
		c.recorder.Eventf(c.nodeRef, v1.EventTypeWarning, "MissingClusterDNS", nodeErrorMsg)
		c.recorder.Eventf(pod, v1.EventTypeWarning, "MissingClusterDNS", "pod: %q. %s", format.Pod(pod), nodeErrorMsg)
		// Fallback to DNSDefault.
		fallthrough
	case podDNSHost:
		// When the kubelet --resolv-conf flag is set to the empty string, use
		// DNS settings that override the docker default (which is to use
		// /etc/resolv.conf) and effectively disable DNS lookups. According to
		// the bind documentation, the behavior of the DNS client library when
		// "nameservers" are not specified is to "use the nameserver on the
		// local machine". A nameserver setting of localhost is equivalent to
		// this documented behavior.
		if c.ResolverConfig == "" {
			for _, nodeIP := range c.nodeIPs {
				if utilnet.IsIPv6(nodeIP) {
					dnsConfig.Servers = append(dnsConfig.Servers, "::1")
				} else {
					dnsConfig.Servers = append(dnsConfig.Servers, "127.0.0.1")
				}
			}
			if len(dnsConfig.Servers) == 0 {
				dnsConfig.Servers = append(dnsConfig.Servers, "127.0.0.1")
			}
			dnsConfig.Searches = []string{"."}
		}
	}

	if pod.Spec.DNSConfig != nil {
		dnsConfig = appendDNSConfig(dnsConfig, pod.Spec.DNSConfig)
	}
	return c.formDNSConfigFitsLimits(dnsConfig, pod), nil
}
```

最终，kubelet通过调用cri运行时创建带有dns配置的容器。
