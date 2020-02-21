---
title: 使用kubeadm搭建一个完整的kubernetes集群
date: 2020-01-20 16:07:04
tags:
---

# 环境准备

本次部署一共三个节点，一个master，两个worker，都是 Ubuntu16.04 的虚拟机。
- master ： 172.31.0.57
- worker01 ： 172.31.0.32
- worker02 ： 172.31.0.53

## 配置

- 2核CPU
- 8G内存
- 40G系统盘+100G数据盘（数据盘用作ceph osd）
- Ubuntu16.04
- 内网互通

# 部署流程

1. 在所有节点安装 Docker 和 kubeadm
2. 部署 Kubernetes Master
3. 部署容器网络插件
4. 部署 Kubernetes Worker
5. 通过 Taint 调整 Master 执行 Pod 的策略
6. 部署 Dashboard 可视化插件（可选）
7. 部署容器存储插件

# 安装 Docker 和 kubeadm

> 备注：本次部署都是在root用户下进行操作
>
> 这一步的所有操作在所有节点上都要执行

```
root@master:~# apt-get update && apt-get install -y apt-transport-https
root@master:~# curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
root@master:~# vi /etc/apt/sources.list.d/kubernetes.list
# 添加以下内容，然后保存退出，使用阿里的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
root@master:~# apt update
root@master:~# apt -y install docker.io kubeadm
```
在安装 kubeadm 的过程中，kubeadm 和 kubelet、kubectl、kubernetes-cni 这几个二进制文件都会被安装好。

安装 docker 直接使用 docker.io 的安装源，因为发布的最新的 Docker CE（社区版）往往没有经过 Kubernetes 项目的验证，可能会有兼容性问题。

另外，后续在执行 kubeadm 命令的时候，会进行一系列的检查工作（“preflight”），其中需要禁用虚拟内存（swap）

```
root@master:~# swapoff -a
root@master:~# sed 's/.*swap.*/#&/' /etc/fstab
```



# 部署 Kubernetes 的 Master 节点

首先查看一下我们本次部署的 kubernetes 的版本信息
```
root@master:~# kubeadm config print init-defaults
.......
.......
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.17.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}

```
> 备注：通过配置文件来开启一些实验性功能。

有了这些信息后，接下来编写一个给 kubeadm 用的 YAML 文件（kubeadm.yaml）:

```
root@master:~# vi kubeadm.yaml

apiVersion: kubeadm.k8s.io/v1beta2
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
controllerManager:
    extraArgs:
        horizontal-pod-autoscaler-use-rest-clients: "true"
        horizontal-pod-autoscaler-sync-period: "10s"
        node-monitor-grace-period: "10s"
apiServer:
    extraArgs:
        runtime-config: "api/all=true"
kubernetesVersion: v1.17.0
```
apiVersion、kind、kubernetesVersion 都可以从上面 print 的返回中得到。

另外把 imageRepository 配置为 `registry.aliyuncs.com/google_containers` 阿里的源，因为默认的 `k8s.gcr.io` 因为一些原因访问不了。

`horizontal-pod-autoscaler-use-rest-clients: "true"` 意味着，将来部署的 kube-controller-manager 能够使用自定义资源进行自动水平扩展。


接下来，只需要一句指令，就可以完成 Kubernetes 的部署
```
root@master:~# kubeadm init --config kubeadm.yaml

......
......

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.0.57:6443 --token 1octc9.3uvkgo5vy1sgr7vy \
    --discovery-token-ca-cert-hash sha256:72fc3bf1a6cc0a159a27bf99518c116845b6a8ae05fb8e78cddc4c7156777c10
```
部署完成后，kubeadm 会生成一行指令：
```
kubeadm join 172.31.0.57:6443 --token 1octc9.3uvkgo5vy1sgr7vy \
    --discovery-token-ca-cert-hash sha256:72fc3bf1a6cc0a159a27bf99518c116845b6a8ae05fb8e78cddc4c7156777c10
```
这个命令，就是用来给这个Master节点添加更多的工作节点使用的。

另外，kubeadm 还会提示我们一些配置命令，也要执行一下：

```
root@master:~# mkdir -p $HOME/.kube
root@master:~# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
root@master:~# chown $(id -u):$(id -g) $HOME/.kube/config
```
这样配置的原因是：Kubernetes 集群默认需要加密方式访问。所以将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的.kube 目
录下，kubectl 默认会使用这个目录下的授权信息访问 Kubernetes 集群。


现在，用 `kubectl get` 命令查看当前唯一一个节点的状态：
```
root@master:~# kubectl get nodes
NAME     STATUS     ROLES    AGE     VERSION
master   NotReady   master   2m26s   v1.17.1
```
输出结果中，master 的状态是 NotReady，排查问题，最重要的手段就是用 `kubectl describe` 来查看这个节点（Node）对象的详细信息、状态和事件（Event）。
```
root@master:~# kubectl describe node master

.....
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 19 Jan 2020 11:07:23 +0800   Sun, 19 Jan 2020 11:07:23 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 19 Jan 2020 11:07:23 +0800   Sun, 19 Jan 2020 11:07:23 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 19 Jan 2020 11:07:23 +0800   Sun, 19 Jan 2020 11:07:23 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Sun, 19 Jan 2020 11:07:23 +0800   Sun, 19 Jan 2020 11:07:23 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized

.....
```
从输出中，可以看到原因在于为部署任何网络插件，`network plugin is not ready: cni config uninitialized`。

我们还可以通过 kubectl 查看这个节点上各个系统 Pod 的状态，其中 kube-system 是 Kubernetes 项目预留给系统 Pod 的工作空间。
```
root@master:~# kubectl get pods -n kube-system 
NAME                             READY   STATUS    RESTARTS   AGE
coredns-9d85f5447-fksl4          0/1     Pending   0          6m45s
coredns-9d85f5447-qqkwn          0/1     Pending   0          6m44s
etcd-master                      1/1     Running   0          7m2s
kube-apiserver-master            1/1     Running   0          7m2s
kube-controller-manager-master   1/1     Running   0          7m1s
kube-proxy-h2kdx                 1/1     Running   0          6m45s
kube-scheduler-master            1/1     Running   0          7m1s
```
从输出中可以看到，coredns 的 Pod 处于 Pending 状态，即调度失败，当然也符合预期，因为这个 Master 节点的网络尚未就绪。

# 部署网络插件

本次我们选择 Weave 为网络插件，只需执行一句`kubectl apply`指令
```
root@master:~# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```
部署完，等一两分钟，再次检查pod状态

```
root@master:~# kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-9d85f5447-fksl4          1/1     Running   0          12m
coredns-9d85f5447-qqkwn          1/1     Running   0          12m
etcd-master                      1/1     Running   0          12m
kube-apiserver-master            1/1     Running   0          12m
kube-controller-manager-master   1/1     Running   0          12m
kube-proxy-h2kdx                 1/1     Running   0          12m
kube-scheduler-master            1/1     Running   0          12m
weave-net-xssqj                  2/2     Running   0          95s
```
现在，所有的系统 Pod 都成功启动了，Weave 插件新建了一个名叫 weave-net-xssqj 的 Pod，这就是容器网络插件在每个节点上的控制组件。

# 部署 Kubernetes 的 Worker 节点

Kubernetes 的 Worker 节点和 Master 节点几乎是相同的，它们运行的都是一个 kubelet 组件。区别在于，Master 节点上多运行这 kube-apiserver、kube-scheduler、kube-controller-manager 这三个系统 Pod。

部署 Worker 节点是最简单的，只需要两步即可。

第一步，在所有的 Worker 节点上执行“安装 Docker 和 kubeadm”这一步的操作

第二步，执行部署 Master 节点时生成的`kubeadm join`命令
```
kubeadm join 172.31.0.57:6443 --token 1octc9.3uvkgo5vy1sgr7vy \
    --discovery-token-ca-cert-hash sha256:72fc3bf1a6cc0a159a27bf99518c116845b6a8ae05fb8e78cddc4c7156777c10
```

这时，我们在 Master 节点上查看一下当前集群的节点（node）
```
root@master:~# kubectl get node
NAME       STATUS   ROLES    AGE   VERSION
master     Ready    master   20m   v1.17.1
worker01   Ready    <none>   1m   v1.17.1
worker02   Ready    <none>   1m   v1.17.1

```
可以看到，当前集群有三个节点，并且都是 Ready。

# 通过 Taint 调整 Master 执行 Pod 的策略

> 备注：默认情况下，Master 节点是不允许运行用户的 Pod 的。

原理比较简单：一旦某个节点被打上了一个 Taint，即“有了污点”，那么所有的 Pod 就都不能在这个节点上运行。

通过 `kubectl descirbe` 检查一下 Master 节点的 Taint 字段，
```
root@master:~# kubectl describe node master
Name:               master
Roles:              master
Labels:             ......
Annotations:        ......
CreationTimestamp:  ......
Taints:             node-role.kubernetes.io/master:NoSchedule
......
```
可以看到，Master节点默认加上了一个`node-role.kubernetes.io/master:NoSchedule`这样的Taint，键是`node-role.kubernetes.io/master`，值是`NoSchedule`。

所以，需要将这个Taint删掉才行
```
root@master:~# kubectl taint nodes --all node-role.kubernetes.io/master-
```
我们在`node-role.kubernetes.io/master`这个键后面加上了一个短
横线`-`，这个格式就意味着移除所有以`node-role.kubernetes.io/master`为
键的 Taint。

至此，一个基本完整的 Kubernetes 集群就部署完毕了。

接下来，会再安装一些其他的辅助插件，如 Dashboard 和存储插件。

# 部署 Dashboard 可视化插件（可选）

Dashboard 的部署也很简单，还是通过`kubectl apply`命令
```
root@master:~# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc2/aio/deploy/recommended.yaml
root@master:~# mv recommended.yaml kubernetes-dashboard.yaml
root@master:~# kubectl apply -f kubernetes-dashboard.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```
部署完成后，在kubernetes-dashboard的命名空间下，会新建两个 Pod。

```
root@master:~# kubectl get pod -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-7b64584c5c-w46jk   1/1     Running   0          3m
kubernetes-dashboard-566f567dc7-554zh        1/1     Running   0          3m

```
> 注意：Dashboard 部署完成后，默认只能通过 Proxy 的方式在本地访问。如果想从集群外访问这个 Dashboard 的话，需要用到 Ingress。

# 部署容器存储插件

本次部署选择 Rook 存储插件。
```
root@master:~# kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/common.yaml
root@master:~# kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml
root@master:~# kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
```
部署完成后，可以看到 Rook 在rook-ceph的命名空间下新建了一些 Pod。
```
root@master:~# kubectl get pod -n rook-ceph           
NAME                                                 READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-4t7qj                               3/3     Running     0          4m
csi-cephfsplugin-9db7n                               3/3     Running     0          4m
csi-cephfsplugin-pb8pv                               3/3     Running     0          4m
csi-cephfsplugin-provisioner-8b9d48896-2qk22         4/4     Running     0          4m
csi-cephfsplugin-provisioner-8b9d48896-dgbkm         4/4     Running     1          4m
csi-rbdplugin-4ln69                                  3/3     Running     0          4m
csi-rbdplugin-7qp7d                                  3/3     Running     0          4m
csi-rbdplugin-provisioner-6d465d6c6f-4jsfj           5/5     Running     0          4m
csi-rbdplugin-provisioner-6d465d6c6f-w4gzx           5/5     Running     1          4m
csi-rbdplugin-sqmvz                                  3/3     Running     0          4m
rook-ceph-crashcollector-master-6b8dd5d4fd-thq4w     1/1     Running     0          3m
rook-ceph-crashcollector-worker01-68d69757d7-j9fzs   1/1     Running     0          4m
rook-ceph-crashcollector-worker02-8474bd88d-xftgn    1/1     Running     0          3m
rook-ceph-mgr-a-76c6c49cc9-7tlzb                     1/1     Running     0          3m
rook-ceph-mon-a-59998d787-bdvlv                      1/1     Running     0          4m
rook-ceph-mon-b-6d69bddc95-vvvkq                     1/1     Running     0          4m
rook-ceph-mon-c-776d969f6c-89fhm                     1/1     Running     0          4m
rook-ceph-operator-678887c8d-sm8bq                   1/1     Running     0          3m
rook-ceph-osd-0-78fd8856df-lrfjx                     1/1     Running     0          3m
rook-ceph-osd-1-5cd4b9564f-k55xt                     1/1     Running     0          3m
rook-ceph-osd-2-7c4d99694f-j8phk                     1/1     Running     0          3m
rook-ceph-osd-prepare-master-8f69h                   0/1     Completed   0          3m
rook-ceph-osd-prepare-worker01-pm6cq                 0/1     Completed   0          3m
rook-ceph-osd-prepare-worker02-dg7gb                 0/1     Completed   0          3m
rook-discover-bpnc7                                  1/1     Running     0          4m
rook-discover-bs9hj                                  1/1     Running     0          4m
rook-discover-wwvr2                                  1/1     Running     0          4m

```
rook-ceph-osd-prepare-master-8f69h、rook-ceph-osd-prepare-worker01-pm6cq 、rook-ceph-osd-prepare-worker02-dg7gb  它们的状态是 Completed，这个没有影响，其实它们是一种 job，完成部署 osd 的工作，执行完之后就变成 Completed 了。真正的 osd 的 Pod 是rook-ceph-osd-0-78fd8856df-lrfjx、rook-ceph-osd-1-5cd4b9564f-k55xt、rook-ceph-osd-2-7c4d99694f-j8phk。

另外，默认启动的Ceph集群，是开启Ceph认证的，这样你登陆Ceph组件所在的Pod里，是没法去获取集群状态，以及执行CLI命令，需要部署Ceph toolbox，配置文件如`toolbox.yaml`所示（参考官网）：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-tools
  namespace: rook-ceph
  labels:
    app: rook-ceph-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rook-ceph-tools
  template:
    metadata:
      labels:
        app: rook-ceph-tools
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: rook-ceph-tools
        image: rook/ceph:v1.2.2
        command: ["/tini"]
        args: ["-g", "--", "/usr/local/bin/toolbox.sh"]
        imagePullPolicy: IfNotPresent
        env:
          - name: ROOK_ADMIN_SECRET
            valueFrom:
              secretKeyRef:
                name: rook-ceph-mon
                key: admin-secret
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /dev
            name: dev
          - mountPath: /sys/bus
            name: sysbus
          - mountPath: /lib/modules
            name: libmodules
          - name: mon-endpoint-volume
            mountPath: /etc/rook
      # if hostNetwork: false, the "rbd map" command hangs, see https://github.com/rook/rook/issues/2021
      hostNetwork: true
      volumes:
        - name: dev
          hostPath:
            path: /dev
        - name: sysbus
          hostPath:
            path: /sys/bus
        - name: libmodules
          hostPath:
            path: /lib/modules
        - name: mon-endpoint-volume
          configMap:
            name: rook-ceph-mon-endpoints
            items:
            - key: data
              path: mon-endpoints
```
执行 rook-ceph-tools的 pod
```
root@master:~# kubectl apply -f toolbox.yaml
```
部署完成后，可以看到在rook-ceph的命名空间下多了`rook-ceph-tools-856c5bc6b4-mw7t8` Pod
```
root@master:~# kubectl get pod -n rook-ceph| grep tools
rook-ceph-tools-856c5bc6b4-mw7t8                     1/1     Running     0          3m
```
现在，进入到pod当中，执行`ceph -s`，查看ceph 集群的状态
```
root@master:~# kubectl exec -it rook-ceph-tools-856c5bc6b4-mw7t8  -n rook-ceph bash

[root@worker02 /]# ceph -s
  cluster:
    id:     11436e29-c940-4a4f-9875-c65fac3531c1
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 5h)
    mgr: a(active, since 5m)
    osd: 3 osds: 3 up (since 5m), 3 in (since 5m)
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   3.0 GiB used, 294 GiB / 297 GiB avail
    pgs:     

```
可以看到，ceph集群是HEALTH_OK的。

