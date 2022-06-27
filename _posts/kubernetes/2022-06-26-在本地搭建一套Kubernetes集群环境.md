---
title: 在本地搭建一套kubernetes集群环境
categories:
  - kubernetes
permalink: /:categories/set-up-a-kubernetes-cluster-locally
tags: 
  - kubernetes
  - 云原生之路
---

在本地通过vmware虚拟机搭建一套供学习使用的kubernetes集群。

<!--more-->

> * 实验环境
>    * 用vmware创建的两台2U4G的[openEuler](https://www.openeuler.org/)虚机，系统安装流程这里不再赘述。
>    * 可以先创建一台虚机，中途克隆（在**二、安装软件**步骤后执行）
>    * 我这里用的是openEuler-22.03-LTS，内核版本为5.10.x。

## 一、系统配置

1. 网络这块，我这边使用了路由器的DHCP的静态地址分配功能，也可以给虚拟机设置静态IP；

2. 修改主机名（PS：我这里用master做主机名主要用于区分k8s中master和node）；

    ```shell
    vim /etc/hostname
    # 输入主机名
    master
    ```

3. 根据实际网络情况，在hosts中将要加入集群的主机进行域名映射

    ```shell
    vim /etc/hosts
    # 增加以下内容：
    192.168.1.200 master
    192.168.1.201 node1
    ```

4. 永久关闭swap（cgroup限制的是物理内存，所以会导致容器内存占用上升的时候，使用swap空间，增加磁盘读写，导致磁盘性能下降。故一般会关闭swap空间）

    ```shell
    swapoff -a
    vim /etc/fstab
    # 注释掉SWAP分区项（第三行），即可
    # /dev/mapper/openeuler-root /                       ext4    defaults        1 1
    # UUID=abeefe9a-1e1f-4ce4-bfd7-9df93940d50b /boot                   ext4    defaults        1 2
    # # /dev/mapper/openeuler-swap none                    swap    defaults        0 0  # 注释掉这行

    #刷新swap使之生效
    sysctl -p
    ```

5. 通过命令 `setenforce 0` 关闭selinux （关闭selinux以允许容器访问宿主机的文件系统）

6. 永久关闭防火墙，或者不关闭防火墙但需要打开一些端口（**二选一**）

    ```shell
    systemctl disable firewalld
    systemctl stop firewalld
    ```

    或

    ```shell
    firewall-cmd --zone=public --add-port=80/tcp --permanent
    firewall-cmd --zone=public --add-port=6443/tcp --permanent
    firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
    firewall-cmd --zone=public --add-port=10250-10255/tcp --permanent
    firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --list-ports
    ```

7. 配置流量转发

    ```shell
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    modprobe overlay
    modprobe br_netfilter

    # sysctl params required by setup, params persist across reboots
    cat <<EOF >  /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    EOF

    vim /etc/sysctl.conf
    # 注释掉 net.ipv4.ip_forward = 0

    # Apply sysctl params without reboot
    sysctl -p /etc/sysctl.d/k8s.conf
    sysctl --system
    systemctl daemon-reload
    ```

## 二、安装软件

> 我在本地实现了局域网的科学上网，故使用的都是默认源，可以使用国内镜像源替换。
> * https://download.docker.com/linux/centos/docker-ce.repo  -> http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
> * https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64 -> http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
> * https://packages.cloud.google.com/yum/doc/yum-key.gpg -> http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
> * https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg -> http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

1. 更新docker和k8s的软件源信息

    ```shell
    # docker-ce 源
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

    # 因为用的是openEuler系统，所以$releasever（即系统版本 22.03-LTS）可能在centos的源中无法找到，故锁定系统版本为8。
    # 不锁定版本9的原因是因为docker版本太新，kubernetes暂不支持。
    # 具体支持版本可以查看kubernetes releases的CHANGELOG.md
    sed -i  "s/\$releasever/8/g" /etc/yum.repos.d/docker-ce.repo

    # kubernetes 源
    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=0
    repo_gpgcheck=0
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF

    # 更新yum包的索引
    yum clean all && yum makecache
    yum -y update
    ```

2. 安装docker，以及配置镜像加速

    ```shell
    # 无论是否安装过docker，都执行以下命令，尝试卸载原有docker
    systemctl stop docker && systemctl disable docker
    yum remove docker-ce docker-ce-cli

    # 安装k8s支持的docker版本，我这里选择了`19.03.15`
    yum list docker-ce --showduplicates|sort -r
    yum install -y docker-ce-19.03.15-3.el8 docker-ce-cli-19.03.15-3.el8
    systemctl start docker && systemctl enable docker

    # 配置docker镜像加速（可选）
    touch /etc/docker/daemon.json
    vim /etc/docker/daemon.json
    {
        "registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"] # 可以选用各大云厂商的容器镜像加速服务
    }
    systemctl daemon-reload
    systemctl restart docker

    # 配置containerd
    rm /etc/containerd/config.toml
    systemctl restart containerd
    ```

3. 安装cni所需二进制， [https://github.com/containernetworking/plugins/releases](https://github.com/containernetworking/plugins/releases)

    ```shell
    cd /opt/cni/bin/
    wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
    tar zxvf cni-plugins-linux-amd64-v1.1.1.tgz
    rm cni-plugins-linux-amd64-v1.1.1.tgz
    ```

3. 安装kubeadm，安装kubeadm的命令同时会安装下面的3个二进制

    ```shell
    # kubeadm k8s的配置工具
    # kubelet k8s的核心服务
    # kubectl kubelet的client
    # kubernetes-cni 容器网络接口标准协议
    yum install -y kubeadm
    systemctl daemon-reload
    systemctl start kubelet && systemctl enable kubelet
    ```

4. **自此，可以克隆多份虚拟机做工作节点了，记得改对应的hostname和静态IP。**

## 三、集群安装

1. 由于国内网络环境的缘故，事先把`kubeadm init`需要的镜像拉下来（已配置科学上网的可以跳过）；

    ```shell
    kubeadm config images list
    # 根据输出修改脚本
    vim pullkubeimages.sh
    ```

    内容如下：

    ```shell
    #!/bin/bash

    # 国内地址
    # registry.cn-hangzhou.aliyuncs.com/google_containers
    # registry.cn-beijing.aliyuncs.com/escience/escience-beijing

    KUBE_VERSION=v1.19.0        # 根据上条命令的输出修改版本
    KUBE_PAUSE_VERSION=3.2
    ETCD_VERSION=3.4.13-0
    CORE_DNS_VERSION=1.7.0

    GCR_URL=k8s.gcr.io
    ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

    images=(
            kube-apiserver:${KUBE_VERSION}
            kube-controller-manager:${KUBE_VERSION}
            kube-scheduler:${KUBE_VERSION}
            kube-proxy:${KUBE_VERSION}
            pause:${KUBE_PAUSE_VERSION}
            etcd:${ETCD_VERSION}
            coredns:${CORE_DNS_VERSION}
    )

    for imageName in ${images[@]} ; do
    docker pull $ALIYUN_URL/$imageName
    docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
    docker rmi  $ALIYUN_URL/$imageName
    done
    ```

2. 初始化master节点

    ```shell
    kubeadm init --service-cidr=10.96.0.0/16 --pod-network-cidr=10.244.0.0/16
    ```

    记录输出

    ```
    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 192.168.1.200:6443 --token 5skkx5.vs59od5xuz31u2b5 \
        --discovery-token-ca-cert-hash sha256:e2ee97296fd9c333a119615fab397828bd226a5ef2e24a93bab334d15f60964b
    ```

3. 按照提示配置集群kubeconfig

    ```shell
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    ```

4. 安装网络插件，参考[https://kubernetes.io/docs/concepts/cluster-administration/addons/](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    ```

5. 验证集群安装情况

    ```shell
    [root@master ~]# kubectl get nodes
    NAME     STATUS   ROLES           AGE   VERSION
    master   Ready    control-plane   92s   v1.24.2

    [root@master ~]# kubectl get cs
    Warning: v1 ComponentStatus is deprecated in v1.19+
    NAME                 STATUS    MESSAGE                         ERROR
    controller-manager   Healthy   ok                              
    scheduler            Healthy   ok                              
    etcd-0               Healthy   {"health":"true","reason":""}
    ```

6. 配置node节点，在节点执行第二步的命令`kubeadm join ...`就可以加入集群。

## 四、功能验证

    1. 验证DNS

    ```shell
    [root@master ~]# kubectl run test1 -it --rm --image=busybox:1.27
    If you don't see a command prompt, try pressing enter.
    / # nslookup kubernetes
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      kubernetes
    Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
    ```

2. 部署服务验证

    ```shell
    [root@master ~]# kubectl create deployment nginx --image=nginx
    deployment.apps/nginx created
    [root@master ~]# kubectl expose deployment nginx --port=80 --type=NodePort
    service/nginx exposed
    [root@master ~]# kubectl get svc -owide
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
    kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        30m   <none>
    nginx        NodePort    10.96.131.212   <none>        80:32055/TCP   15s   app=nginx
    [root@master ~]# curl 192.168.1.201:32055
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    ...
    ```