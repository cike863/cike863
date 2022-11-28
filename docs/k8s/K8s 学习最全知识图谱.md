![图片](https://mmbiz.qpic.cn/mmbiz_jpg/yvBJb5Iiafvk3b362srzPuh3gSXbZYEQcJLPJcClaATscGLbIwsrP2bph0Ie5wQG6NOdiaEBicJKDuOP3KiaBE3lRw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

**导读**：Kubernetes 作为云原生时代的“操作系统”，熟悉和使用它是每名用户的必备技能。本篇文章概述了容器服务 Kubernetes 的知识图谱，部分内容参考了网上的知识图谱，旨在帮助用户更好的了解 K8s 的相关知识。



![图片](https://mmbiz.qpic.cn/mmbiz_gif/US10Gcd0tQGY9ddd5GpbmVRuaRfuaESAUBGE7uHX5G0nxxLSub2QTKZdu538V7GaHXS5jsTCebYCUibaHsjg0ow/640?wxfrom=5&wx_lazy=1)

**概述**

容器服务 Kubernetes 知识图谱，部分内容参考网上一知识图谱，更加结合阿里云容器服务。

![图片](https://mmbiz.qpic.cn/mmbiz_png/yvBJb5Iiafvk3b362srzPuh3gSXbZYEQcicvEjDYNYAoljEyCOUBSFn7FSnOtokibjbtLv3jpiajnjqY7CgEQ3ibfRA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

原图 by 杨传胜

原图链接地址

https://www.processon.com/view/link/5ac64532e4b00dc8a02f05eb#map

![图片](https://mmbiz.qpic.cn/mmbiz_gif/US10Gcd0tQGY9ddd5GpbmVRuaRfuaESAUBGE7uHX5G0nxxLSub2QTKZdu538V7GaHXS5jsTCebYCUibaHsjg0ow/640?wxfrom=5&wx_lazy=1)

**知识链接和备注**

### **Docker 原理**

- KVM--> ECS

https://blog.csdn.net/weixin_43695104/article/details/88554443#32_kvm_web_192

- 网络隧道技术-->VPC

https://blog.csdn.net/wangjianno2/article/details/75208036

- NameSpace

https://blog.csdn.net/a352193394/article/details/53344167

备注：Linux 容器中用来实现“隔离”的技术手段：Namespace，Namespace 技术实际上修改了应用进程看待整个计算机的范围，它的访问范围被操作系统做了限制，只能“看到”某些指定的内容。

- CGroup

https://blog.csdn.net/wudongxu/article/details/8474198

备注：Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。

- RootFS(Union FS)

https://coolshell.cn/articles/17061.html

备注：rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。

- windows 2019

备注：windowserver 2019开始支持 namespace

### **容器服务部署**

- Docker Desktop

https://www.docker.com/products/docker-desktop

备注：Mac 机器上强烈建议安装该软件作为学习使用

- kubernetes

http://docs.kubernetes.org.cn/

备注：kubernetes 集群，aliyun容器服务支持

- DashBoard

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

备注：kubernetes 集群的图形界面管理工具，容器服务控制台整合了该应用并扩展

- EasyPack

https://github.com/liumiaocn/easypack

备注：一批部署 kubernetes 等集群的脚本集合

- minikube

https://kubernetes.io/docs/tasks/tools/install-minikube/ 

备注：mini 新 K8s

### 

### **工具组件**

- kubectl

http://docs.kubernetes.org.cn/61.html

备注：kubectl 用于运行 Kubernetes 集群命令的管理工具

- kubeadm

https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/Kubernetes

备注：官方提供的用于快速安装配置 Kubernetes 集群的工具

- Helm

备注：类似 rpm，yum，是 K8s 用于安装组件（软件包：chart）的工具

- APP Hub

https://developer.aliyun.com/hub

备注：在开放云原生应用中心当中，所有默认的 Helm Charts（Helm 格式的应用），都定时同步自 Helm Hub 北美官方站并托管在 Github 上。在这个过程中，云原生应用中心会自动对同步过来的所有 Charts 进行“本地化”操作。

- CFSSL

https://github.com/cloudflare/cfssl

备注：CFSSL 是开源的一款 PKI/TLS 工具，常用于 K8s 证书制作

### **镜像仓库**

- aliyun 私有镜像仓库

https://cr.console.aliyun.com/aliyun 

备注：推出的镜像仓库，建议采用企业版

- 云效配置镜像仓库

https://cn.aliyun.com/product/yunxiao

备注：云效企业设置，配置支持从阿里云私有镜像仓库拉取镜像

- Harbor 镜像仓库

https://goharbor.io

备注：开源免费的存储和分发Docker镜像的企业级Registry服务器

### **组件**

- kube-apiserver(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

备注：在 generic server 上封装的一层官方默认的 apiserver(static pod)

- etcd(Master)

https://etcd.io

备注：类 zk 基于 Raft 协议的实现，启动进程

- Kube-scheduler(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/

备注：负责 pod 分布到 Node 上的调度器 (static pod)

- kube-controller-manager(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/

备注：Deployment 等基础对象的控制器 (static pod)

- cloud-controller-manager(Master)

https://kubernetes.io/docs/reference/command-line-tools-reference/cloud-controller-manager/

备注：用于云资源使用的控制器，是云服务进行集成的控制器 (Daemonset)

- kubelet(Node)

https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/

备注：与 Master 通信,对 worker(Node) 进行生命周期管理

- kube-proxy(Node)

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/

备注：节点上运行的网络代理 (Daemonset)

- containner runtime(Node)

备注：CRI 接口

- DNS

https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

备注：aliyun容器服务采用 CoreDNS(deployment)

- Ingress controller

https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

备注：aliyun容器服务采用nginx ingress controller, 可以作为 https 服务的统一路由(deployment)

- Heapster & influxdb 

备注：监控数据采集与存储用的时序数据库(Deployment)

- Federation

https://kubernetes.io/docs/concepts/cluster-administration/federation/

备注：集群联盟，实现高可用，同步资源等

- kube-flannel

备注：官方网络插件，aliyun 另外提供了自己开发的 Terway 组件(daemonset)

- logtail

https://help.aliyun.com/document_detail/28979.html?spm=a2c4g.11186623.6.595.439d7218wQhzsH

备注：aliyun 日志采集组件 (daemonset)

### **基础对象**

- POD

http://docs.kubernetes.org.cn/312.html  

容器组，运行应用容器基本单位，kubectl get pods 

- Node

http://docs.kubernetes.org.cn/304.html

集群节点服务器，Kubernetes中的工作节点。

- NameSpace

http://docs.kubernetes.org.cn/242.html

备注：用以区分和隔离应用

- Deployement

http://docs.kubernetes.org.cn/317.html

备注：无状态部署，最常用部署配置

- Daemonset

https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

备注：类似守护进程

- StatefulSet

http://docs.kubernetes.org.cn/443.html

备注：有状态部署

- Job & CronJob

https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/

备注：调度任务

- Static POD

https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/

备注：静态 pod 配置，yaml 位于 Master

- HPA

https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

备注：水平伸缩调度器

- Service

https://kubernetes.io/docs/concepts/services-networking/service/

备注：服务暴露配置，包括 Cluster,NodePort,SLB 等

- Ingress

https://www.kubernetes.org.cn/1885.html

备注：路由，阿里云默认提供 nginx ingress

- Secret

https://kubernetes.io/docs/concepts/configuration/secret/

备注：保密字典，包括 tls,私有仓库密钥，Opaque 几种

- ServiceAccount

https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/

备注：用于资源对象的账号，比如给一个 Namespace 授予某私有镜像访问权限

- RBAC

https://kubernetes.io/docs/reference/access-authn-authz/rbac/

备注：K8s 基于角色的访问控制，role,rolebinding

- Volume

https://kubernetes.io/docs/concepts/storage/volumes/

备注：映射磁盘

- Storge Class

https://kubernetes.io/docs/concepts/storage/storage-classes/

- CutomResourceDefinition

备注：自定义扩展资源

### **插件扩展**

- CNI(Falnnel/Terway)

https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/

备注：容器网络接口

- FlexVolume

https://github.com/fstab/cifs

备注：开源 Volume 实现插件，阿里云使用中

- Cloud Provider

备注：云服务供应接口

### **容器服务优化-最佳实践**

- Master 选型及磁盘规格

[1] https://yq.aliyun.com/articles/599169?spm=5176.11065265.1996646101.searchclickresult.7bea1a8bgCTYH7

[2] https://yq.aliyun.com/articles/621108?

- 网络选择

https://yq.aliyun.com/articles/594943?

- Worker 节点选型

https://yq.aliyun.com/articles/602932?spm=a2c4e

- Ingress Controller 独立部署

- Master 变配

https://help.aliyun.com/document_detail/123661.html?spm=5176.10695662.1996646101.searchclickresult.20d0328c6WG7jc

- 节点变配或重启、摘除、加入
- 基础镜像开发
- Service 与 SLB 结合



- 集群审计

https://help.aliyun.com/document_detail/91406.html?spm=5176.10695662.1996646101.searchclickresult.45266c92kGHQrP

- Deployment实现分批发布
- StatefulSet 分批发布

https://yq.aliyun.com/articles/622898?spm=a2c4e.11155435.0.0.1b8e3312bSGmSe

- 堡垒机上按照应用设置权限

https://yq.aliyun.com/articles/715809

- Pod 均匀分布部署

https://yq.aliyun.com/articles/715808

- 应用优雅下线，优雅退出

- ApiServer 访问

- 监控

  

### **服务治理**

- Istio

https://istio.io

备注：当前最流行的网格服务架构，aliyun支持

- Linkerd

https://linkerd.io/2/overview/

备注：最早提出网格服务公司的产品 

- 云效

https://www.aliyun.com/product/yunxiao

备注：支持容器服务 K8s 的 CI/CD 阿里云上产

- Jenkins

https://jenkins.io/zh/

备注：著名的最常用的 CI/CD 产品，容器服务由一键安装产品

『本文转载自阿里云开发者社区』

原文链接：

https://developer.aliyun.com/article/715805?utm_content=g_1000073582