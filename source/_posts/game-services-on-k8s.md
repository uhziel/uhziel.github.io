---
title: 以云原生的方式部署游戏服务到 k8s 集群
date: 2023-02-21 15:03:29
tags:
  - kubernetes
---

在多年游戏开发过程中，深刻体验到传统部署方式的不灵活。

期望通过应用已经成熟的云原生技术，在尽量减少项目代码修改的前提下，实现部署的简化。

选择的环境:

* 内网:节点的操作系统 CentOS 8.5.2111、kubernetes 发行版 rancher rke2。
* 现网:腾讯云 tke。

核心目标:

* 通过扩展 kubernetes 添加新 CRD 实现 Operator，让我们可通过描述游戏各服务完成声明式部署。
* 现网中，利用云的弹性，实现云资源的按需申请，达到降本增效的目的。
* 内网中，应用 GitOps 让项目开发可以自服务创建测试环境。实现 fork 示例仓库(包含游戏服务、redis、mysql 声明式定义)后，自动拥有一个完整的环境，暴露出的服务可通过内网域名访问(示例:redis.zhulei.rke2.example.com)。
* 尽量消除内网/现网环境的不一致，让项目开发在内网环境就可以完成交付测试。
* 交付给现网的不止是声明式部署游戏服务，还应包括腾讯云基础设施的声明式申请、日志聚合、跟随游戏版本升级的指标监控等。

## 设计方案

采用 kubernetes 的 [Operator 模式](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)，使用 kubebuilder 扩展 k8s 的能力制作我们自己的 operator，实现 CRD GameZone。

示例

```yaml
apiVersion: apps.example.com/v1alpha1
kind: GameZone
metadata:
 name: zone1
spec:
 gameZone:
   zoneId: 101
   dbMigraterImage: registry.example.com/project1/dbmigrater:0.1.0
   defaultImage: registry.example.com/project1/service:0.1.0
   serviceConfig:
     - type: game
       id: 1
       image: registry.example.com/project1/service:0.2.0 # 优先级高于 defaultImage
     - type: db
       id: 1
     - type: gateway
       id: 1
       endId: 2
     - type: auth
       id: 2
   redisConfig:
     ip: redis
     port: 6379
   mysqlConfig:
     ip: 192.168.1.100
     port: 3306
     user: username1
     pwd: password1
     database: project1
 artifacts: # 制品配置。实现文件资源、配置的获取。
   - path: /app/artifacts
     s3:
       url: https://cos.ap-beijing.myqcloud.com
       accessKey: AKIDajocbuwoqooojfqoqujjofquorqou
       secretKey: 123456789012345678901234567890
       path: example-1234567890/project1/0.1.0/resources/
   - path: /app/artifacts/config
     subPath: config # 只有下面来源子目录 subPath 里的内容会放到 path 里。
     git:
       url: https://gitlab.example.com/prject1/config.git
       branch: main
```

（一）service 镜像

制作出的容器镜像包含所有服务的二进制可执行文件和库。不带文件资源、配置等制品数据。

通过传入的参数(type: game)控制容器镜像启动哪个服务。

远程访问使用 `kubectl exec -it <pod-name>`，可以获得类似 ssh 的效果。

对 k8s 集群外暴露服务：

* 内网：通过 MetalLB 使用 service LoadBalancer 实现。
* 外网：通过 clb 使用 service LoadBalancer 实现。

（二）dbmigrater 镜像

operator 会自动启动 dbmigrater 实现数据库表样式的升级。

（三）gameZone 游戏大区配置

通过 defaultImage 控制使用的程序版本。 通过 serviceConfig 下的 image 自定义单个服务使用的程序版本。用来支持修复 bug 等需求。

（四）artifacts 制品配置

用来实现文件资源、配置等制品数据的获取。

artifacts 里支持配置多个 artifact。从上到下依次拉取，支持目录的覆盖，后面的可以覆盖前面的内容，以实现散乱各处文件的融合。
通过文件共享(smb)的方式共享，方便测试人员临时修改文件。

制品来源途径：

* p4。目前公司项目的大部分数据都是来自 p4，这个是必须要支持的。
* git。它可以方便开发人员、测试人员建立自己的仓库准备一些文件覆盖前面的 artifact。subPath 是为了解决 git 只能拉取整个仓库但有时只需部分文件夹的情况。
* s3。外网部署无法访问代码仓库，可以通过它实现获取。

（五）实际效果

* 当集群中创建 CR GameZone 后，自动完成文件资源、配置的拉取，数据库样式表的升级、服务的部署。很方便就可以获得一个可运行的游戏大区。
* 当删除该 CR GameZone 后，会自动销毁该游戏大区，释放占用的资源。
* 当修改 CR GameZone 内容后，会自动让集群当前状态反应我们在 CR GameZone 中的描述。比如，添加一个 game-2、修改 mysql 配置、升级 service 程序容器镜像的版本。
* 上述所有功能都不用修改游戏项目的代码就能得到支持。
* 部署时，配置变得简单。不用再配置服务 ip 地址，也不用再担心“传统部署中端口冲突”的问题。

## 网络性能测试

通过之前的《{% post_link distributed-code-analysis-on-k8s %}》里的试验，看出容器化对 cpu 的消耗并不大。项目中的服务都是计算密集型，对 disk io 要求并不高。

需要注意的是容器化后的网络性能表现。下面先针对内网 rancher rke2 集群环境进行验证，再针对腾讯云 tke 环境验证。

### k8s网络模型 (The Kubernetes network model)

k8s 网络模型定义了这么一个扁平网络：

* 每个 pod 拥有它自己的 ip 地址
* pod 内的所有容器共享同一个 pod ip 地址，容器间可以通过 loopback 自由通信
* pod 间可以通过 pod id 地址通信（无需 NAT）

pod 对应传统的虚拟机，每个虚拟机也有它自己的 ip 地址；容器对应虚拟机内的进程，进程间也是可以通过 loopback 通信。这种向后兼容的网络模型，很方便我们把传统的应用程序迁移到 k8s。更多的信息请看官方的[The Kubernetes network model](https://kubernetes.io/docs/concepts/services-networking/#the-kubernetes-network-model)。

为了实现这个网络模型，我们的 k8s 集群需要使用 CNI 插件。

### CNI 插件

rancher rke2 默认使用的 CNI 插件是 Canal，它的网络性能并不突出，胜在能适应各种底层网络环境。更多信息请看[Container Network Interface (CNI) Providers](https://rancher.com/docs/rancher/v2.6/en/faq/networking/cni-providers/#rke-kubernetes-clusters)。

[市场上的 CNI 插件插件众多](https://landscape.cncf.io/card-mode?category=cloud-native-network)，基于性能、成熟度、受欢迎程度，选出两个候选的 CNI 插件：

* [calico](https://www.tigera.io/project-calico/)。可以配置的选项众多，包括允许使用 ebpf 作为数据平面加速网络。
* [cilium](https://cilium.io/)。基于 ebpf，拥有良好的网络性能。

cilium 做了一个网络性能测试分析 [CNI Performance Benchmark](https://docs.cilium.io/en/stable/operations/performance/benchmark/)，比较非容器化、cilium、calico 的网络性能。从上面看，cilium ebpf、calico ebpf 都拥有接近非容器化时的性能，我们可以选择使用它们。

### Service Mesh

Service Mesh 使用 sidecar 来截获并转发 services 间的流量，实现流量管理(traffic management，比如超时、断线重试)、观测(observability)等。

每个 pod 内部都会有个 proxy 容器作为 sidecar。一个请求/回应流程，必须额外经过4次 proxy。导致延迟必然出现比较大的增长。
在被用在 http 服务上时，这个延迟可以被接受；但被用在游戏服务上时，这个延迟是否可以接受，还需要验证。

所有 service mesh 都不会处理 udp 流量。

[市面上的 Service Mesh 也众多](https://landscape.cncf.io/card-mode?category=service-mesh)，基于成熟度、受欢迎程度、性能，选出两个候选的

* [istio](https://istio.io/)。目前市场上的主流，使用的 sidecar proxy 是 envoy，拥有最全的功能。市面上其他 service mesh 也大都使用的是 envoy。
* [linkerd](https://linkerd.io/)。最先提出 service mesh 这个概念的就是它，使用的 sidecar proxy 是用 rust 编写的 linkerd2-proxy，功能相对 istio 少但是网络性能要比 istio 高。

linkerd 也类似的做了一个网络性能测试分析 [Benchmarking Linkerd and Istio: 2021 Redux](https://linkerd.io/2021/11/29/linkerd-vs-istio-benchmarks-2021/)。它比较了无 service mesh、istio、linkerd 三种情况下的网络性能，只要使用了 service mesh 延迟就有比较大的增长，istio 增长的更多。

### 确定网络性能测试内容

游戏项目目前的现状

* tcp长连接。主要是内部进程间通信使用。
* tcp短连接。主要是内部进程调用外部 http api。

从上面看，需要测试的网络性能指标是：

* tcp 的 Request/Response Rate
* tcp 的 Connection Rate

从 [CNI Performance Benchmark](https://docs.cilium.io/en/stable/operations/performance/benchmark/) 看 cilium ebpf、calico ebpf 都能取得不错的效果。

我们剩下的就是以游戏项目实际表现进行测试。

### 内网 rancher rke2 中游戏项目实际表现

硬件与系统

| ip | os | cpu | mem | ssd |
| ------------- | ----------------------- | ---------------------------- | ---- | ------ |
| 172.17.50.214 | CentOS 8.5.2111(Vmware) | Intel Xeon E5-2630 v3 8cores | 16GB | 150GiB |
| 172.17.50.215 | CentOS 8.5.2111(Vmware) | Intel Xeon E5-2630 v3 8cores | 16GB | 150GiB |

相关软件/镜像使用的版本

| 软件 | 版本 |
| ---- | ---- |
| redis | 3.2.12 |
| mysql | 5.7.27 |
| rke2 | v1.23.7+rke2r2 |
| kubernetes | v1.23.7 |
| calico | v3.23.1 |
| cilium | v1.11.5 |
| istio | 1.14.1 |
| linkerd | 2.11.2 |

|  环境  | QPS1 | QPS2 | QPS3 | QPS_avg |
|--------|--------------|--------------|--------------|------------|
| 非容器化 | 2832.86 | 3125.00 | 3311.26 | 3089.7 |
| cni:calico 服务网格:none | 2114.16 | 2386.63 | 2652.52 | 2384.47 |
| cni:calico 服务网格:istio | 1278.77 | 1221.00 | 1240.69 | 1246.82 |
| cni:calico 服务网格:linkerd | 1394.70 | 1414.43 | 1336.90 | 1382.01 |
| cni:cilium 服务网格:none | 2857.14 | 2597.40 | 2673.80 | 2709.45 |
| cni:cilium 服务网格:istio | 1432.66 | 1273.89 | 1424.50 | 1377.02 |
| cni:cilium 服务网格:linkerd | | | | 无法安装成功 |

最接近非容器化的是"cni:cilium 服务网格:none"，QPS为非容器化时的 88%。

而且 cilium 因为测试机 linux kernel 没有更新到足够新，部分 ebpf 相关加速功能还没开启，到腾讯云上可能会有更好表现。

选择：

* cni 插件我们没有用到高级特性，使用上 calico、cilium 其实是可以随时互换的。calico 是支持 windows 的，bgp 支持度更成熟；但是腾讯云 tke 有使用 cilium。先使用 cilium 和腾讯云尽量保持一致。
* 项目中的服务使用的是自定义协议，在服务网格中无法获得像 http、grpc 那样的完整能力；而且服务网格的加入后，会导致 QPS 比较大的降低。所以，先不使用服务网格。

内网环境网络性能不存在问题。最终确定使用 cni: cilium + 不使用服务网格。

注：后续需使用服务网格时，可考虑 [Aeraki Mesh - Manage Any Layer-7 Protocols in a Service Mesh](https://www.aeraki.net/) 来让服务网格支持我们的自定义协议。

## 腾讯云 tke 中游戏项目实际表现

测试环境为下面几种(被测端：腾讯云机型 S6.LARGE16 4核16G；施压端：始终为外部机器，它不存在性能瓶颈。)

* tke。腾讯云 k8s 集群 tke 中部署服务，服务最后放在容器中。
* cvm。直接在组成 tke 的节点上部署服务，节点实际是虚拟机cvm，服务最后直接放在 cvm 中。

| 测试环境 | 最大吞吐量1    | cpu% 1                   | 最大吞吐量2     | cpu% 2                   | 最大吞吐量3    | cpu% 3                   | 最大吞吐量avg   |
|------|-----------|--------------------------|------------|--------------------------|-----------|--------------------------|------------|
| cvm  | 284459.05 | third 75.3% gmtool 85.7% | 288369.11  | third 74.9% gmtool 90.3% | 291866.65 | third 72.9% gmtool 91.2% | 288231.60  |
| tke  | 304372.82 | third 76.4% gmtool 87.8% | 297885.02  | third 75.9% gmtool 83.4% | 298943.73 | third 75.1% gmtool 86.2% | 300400.52  |

数据的解读：

* 放到 tke 中后，QPS 反而要比 cvm 略高 4.2%。这点可能是腾讯对容器有优化造成的。

腾讯云 tke 环境，网络性能不存在问题。

## 总结

通过上面的验证，确认方案的可行性。

最终，通过实际的开发和完善，完成上面的核心目标，让我们得到一些实际的应用场景。比如：

* 公有云k8s上的大规模压力测试，按需申请资源。可以在 CR GameZone 上配置各服务需要的数量，会最终向共有云申请合适的资源；当不用测试时，删除 CR GameZone 即可。可以方便测试不同的云虚拟机机型，选择最适合自身服务的机型。可以方便调整各服务数量，对应的云资源也会自动添加/释放。
* 机器人放入 k8s 中。可以方便的提供更大规模的压力，按需弹性的申请/释放资源。
* 内网中，应用 GitOps 让项目开发可以自服务创建测试环境。实现 fork 示例仓库(包含游戏服务、redis、mysql 声明式定义)后，自动拥有一个完整的环境，暴露出的服务可通过内网域名访问(示例:redis.zhulei.rke2.example.com)。
