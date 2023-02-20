---
title: 构建基于 k8s 的分布式静态代码分析平台
tags:
  - kubernetes
date: 2022-03-28 21:15:30
---


## 一、面临的问题

在近2年的静态代码分析开发时间里，我们试用过市面上的静态代码分析平台 sonarqube、pvs-studio、codechecker、codeclimate、腾讯 CodeCC。也自研了一个简单的静态代码分析平台。

但它们各自有一些或多或少的问题。而对于公司内部大型 cpp 项目来说，最关键的分析速度，更是没有一个能让人接受。因为它们全都是单机器分析，没法利用分布式水平扩充分析能力。这导致我们必须开发一款新平台。

通过历史经验，整理出新平台需要实现的核心目标：

1. 分析速度能够极大提升。通过分布式实现。
2. 方便查看检查出的问题。尤其是：
    * 支持只查看增量问题。对于老项目初次接入来说，这点尤其重要。
    * 能定位到问题作者。方便问题的处理和管理。
3. 接入新项目应该容易。
4. svn, git, p4 应得到支持。新加其他源代码管理工具支持不应该复杂。
5. 易于接入新的静态代码分析工具。以插件的形式先支持 cppcheck、cpplint，其他的工具的插件也应容易编写。
6. 静态代码分析工具插件需易于升级。
    * 升级分析工具官方版本、根据项目需求修改分析工具代码以定制检查逻辑，都会导致升级。
    * 需保证插件老版本和新版本能共存。好处：项目可同时使用老/新版本，等稳定，再切换到只新版本。
    * 发现新版本有问题，回滚应容易。

## 二、技术栈选择

分布式方式，大概是这么几个：

* kubernetes。它是一个容器编排系统。虽然最初是作为一个服务部署平台被推出，但随着 Job 概念的加入，也可以被当作一个分布式批处理平台。
* 改造现有分布式编译系统。比如 Incredibuild、distcc、fastbuild。pvs-studio 是支持借用 Incredibuild 进行分布式分析的。简单看下各自的文档，Incredibuild、fastbuild 具有改造的可能性，但是并不容易，优先考虑 kubernetes。
* 自研。这个耗时最多，只有前面两种都不满足需求时，才考虑。

之前在调研 cppcheck v1.90 切换到 v2.3 分析变非常慢时，有使用 linux perf 生成火焰图查看原因，同时也看到了其实静态代码分析对 disk io 的要求并不高，主要耗时在生成 ast 和检查项检查上。可以考虑用 nfs 做为存储。

之前为开源的 codechecker 贡献过一个特性 [支持接入 cpplint 的输出](https://github.com/Ericsson/codechecker/pull/3248)。类似的，也可以让 codechecker 支持我们自定义的问题格式。可以选择 codechecker 作为我们的问题查看平台，它的分析查看功能大体符合我们的需求，只需我们二次开发添加它缺失的特性即可。

codeclimate 虽然自身不开源，但是它的插件规范 [Code Climate Engine Specification](https://github.com/codeclimate/platform/blob/master/spec/analyzers/SPEC.md) 却可以借鉴。基于它规范的第三方插件很多都是开源的，可以先改造第三方的 [codeclimate-cppcheck](https://github.com/antiagainst/codeclimate-cppcheck) 进行试验，节省原型时间花费。如果可能，尽量兼容它的插件规范。

总结下最初设想的原型技术栈：

* kubernetes Job + redis work queue 为我们提供基础的分布式支持，使用 kubebuilder 扩展 k8s 以支持我们定义的 CRD CodeAnalysisJob
* 插件规范借鉴 codeclimate
* codechecker 作为问题查看平台

## 三、设计方案

实现
（一）分析流程图
![distributed-code-analysis-flowchart](images/distributed-code-analysis-flowchart.drawio.svg)


（二）使用 workqueue 来完成任务的分发图
![distributed-code-analysis-workqueue](images/distributed-code-analysis-workqueue.drawio.svg)

接口

（一）kubernetes CRD

```yaml
apiVersion: batch.example.com/v1alpha1
kind: CodeAnalysisJob
metadata:
  name: project1
spec:
  repo:
    type: git
    url: https://github.com/uhziel/demo-cpp-cmake.git
    branch: main
  config:
    version: "1"
    fileLists:
      debug:
        type: txt
        content: |-
          /code/test1.cpp
      compiled-sources:
        type: shell
        content: |-
          cd /code && mkdir -p build && cd build
          cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
          jq -r "[.[] | .file ] | unique | .[]" compile_commands.json > /tmp/filelist
    analyses:
      - type: cppcheck
        name: cppcheck-1.90
        channel: 0.1.0-1.90
        enabled: true
        config: |-
          {
            "project": "/code/build/compile_commands.json",
            "inline_suppr": true,
            "enable": "all",
            "inconclusive": true,
            "suppressions_list": "/dataFiles/suppressions.txt",
            "stds": ["c++03"],
            "jobs": 14
          }
        fileList: client
      - type: cppcheck
        name: cppcheck-2.3
        channel: 0.1.0-2.3
        enabled: false
        config: |-
          {
            "jobs": 7
          }
        fileList: debug
        workerNum: 18
    outputs:
      - type: codechecker
        env:
        - name: CODECHECKER_WEB_URL
          value: http://codechecker.example.com/project1
  dataFiles:
    suppressions.txt: |-
      noExplicitConstructor:LogServer/logserver/OSSLoger.h
      unusedFunction
```

（二）配置文件 .codeanalysis/config

如果不填写 CodeAnalysisJob CRD 的 spec.config，则从项目代码仓库里的文件 .codeanalysis/config 读取 config。

让项目用户在自己的仓库里就可以完成对静态代码分析的所有配置。

（三）分析插件规格说明

主要参考了 [Code Climate Engine Specification](https://github.com/codeclimate/platform/blob/master/spec/analyzers/SPEC.md)，但是细节有些变化。主要内容：

* 分析插件为 docker 容器。传入源代码和分析的配置，在 stdout 上返回分析结果。只是一个无状态的只读程序，方便分布式部署。
* input
  * /code 以只读方式挂载项目代码
  * /config.json 以只读方式挂载该插件的配置
    * "config" 属性来自配置文件 .codeanalysis/config 中 analyses 的"config"部分
    * "include_paths" 为需要被分析的源代码文件列表
* output
  * stdout 以流的方式输出结构性 json 的 [issues](https://github.com/codeclimate/platform/blob/master/spec/analyzers/SPEC.md#issues)
  * stderr 可以放非结构性的日志，方便查看分析进度、错误日志
  * 找出问题时 exit code 应为 0；只有无法分析时，才返回非 0

下面为分布式分析而加的规则

* image 读取环境变量 CODE_ANALYSIS_WORK_QUEUE_URL、CODE_ANALYSIS_WORK_QUEUE_NAME，如果有值，则从中读取需被分析的文件路径，进入 worker 模式；否则按照正常分析代码。
* worker 模式就是不停的从 work queue 中获取需要分析的文件路径来分析，直到工作队列为空才停止。

## 四、原型验证

测试环境：

* 2 节点的裸金属 kubernetes 集群。
* 节点配置如下：
  * cpu: cpu Intel(R) Core(TM) i7-10700K 8核16线程
  * mem: 32G
  * disk: ssd
  * os: centos 8.4
* code: 内部大型 cpp 项目 project1 的一个分支

在验证时，按照顺序最应该关注的是这些核心关注点：

1. nfs能否承受压力。如果不能，是否能找到替代存储方案。只用安装 nfs 服务即可验证。
2. 容器化的开销。
3. 最终分析耗时是否能有效减少。
4. 正确性。分布式后，能否检查出和原来没分布式时同样的问题。

### 关注点1 nfs 能否承受压力

测试：

* 在文件系统 local 和 nfs 上，测试读性能。“读”指的是，code 中所有文件各读一遍。
* 在文件系统 local 和 nfs 上，测试写性能。“写”指的是，拉取 code 的所有文件。

数据：

| 行为 | 文件系统 | 时间 |
| ---- | -------- | ---- |
| 读 | local | 3m17.398s |
| 读 | nfs | 4m50.395s |
| 写 | local | 1m51.062s |
| 写 | nfs | 134m59.515s |
| 写 | nfs(配置 sync 改为 async)  | 28m27.646s |

结论和解决方法：

* 读不存在问题，但 nfs 在写上没法接受。拉代码这个操作是没法分布式的，增加的几十倍时间完全不可接受。之前设想的完全使用 nfs 看来是不可能。
* 看下是否能支持先拉取代码到 local，再通过 nfs 读这份 local 的数据。最终通过试验，kubernetes 可以变通支持这种操作。问题得到解决。

### 关注点2 容器化的开销

编译 cppcheck 的环境：

* centos8：[Dockerfile.centos8](https://github.com/uhziel/docker-cppcheck/blob/main/Dockerfile.centos8)
  * gcc 版本: gcc version 8.4.1
* debian bullseye：[Dockerfile](https://github.com/uhziel/docker-cppcheck/blob/main/Dockerfile)
  * gcc 版本: gcc version 10.2.1 20210110 (Debian 10.2.1-6)

| 编译环境 | 运行时环境 | 耗时 |
| -------- | --------- | ---- |
| centos8 | 宿主：centos8.4 裸金属 | 662m40.691s |
| centos8 | 宿主：centos8.4 容器：centos8 | 665m28.089s |
| debian bullseye | 宿主：centos8.4 容器：debian bullseye | 598m56.334s |

结论：

1. 容器化的开销并不大。时间仅比裸金属多 0.45 %，完全在可接受范围。
2. 使用 debian bullseye 编译出的 cppcheck 速度明显要快。看来应该是 gcc 10 带来的效果，没有太深究。最终选择 debian bullseye 容器作为分析插件的运行环境。

### 关注点3 最终分析耗时是否能有效减少

事情并不是很顺利，分布到集群中 2 个节点后，速度反而要比单机**慢1倍**。

首先，想到的是 nfs 性能问题。试验如下：

| 用例 | 时间 |
| ---- | ---- |
| 2节点 local 存储 | 19h |
| 2节点 nfs 存储 | 19h |

!!! tip
    使用 local 存储进行分布式测试的技巧是：本地各节点各自准备一份同样的代码，只是原型测试，手工操作是可以接受的。

看着两者没有差别，这个结果和前面“关注点1”的读测试时长也对应得上。排除 nfs 的问题。

再看下分析插件自身的耗时在哪里。在分析插件中加日志，重新跑一次，得到下面的分析插件耗时（时间单位：s）：

| 文件 | 阶段1 | 执行cppcheck | 阶段3+4+5 | 阶段all |
| ---- | ---- | ------------------ | ---- | ---- |
| file1.cpp | 0.0005 | 538.97807 | 0.00115| 538.97968 |
| 所有 cpp | 7.4366 | 1944778.2 | 82.67634| 1944868.4 |

耗时绝大部分都在执行 cppcheck 进行分析上。通过 linux perf 得知 cppcheck 进程分析外开销不少，主要是解析编译数据库上耗时太多。

而最初，本人是把 project1 的近1万 个 cpp 文件分割为近 1 万份，各启动一个 cppcheck 进行分析。导致分析外的开销太大。

解决方案显然也就是尽量减少分割。最终分割为两份后的结果如下：

| 用例 | 时间 |
| ---- | ---- |
| 单机 | 10h |
| 分布式 2 节点 | 5h3m |

结论：差不多减少 50%，效果不错。后面随着集群节点的增多，虽然效果会减弱，但也不会差太多。

### 关注点4 正确性

针对 project1 使用以前的工具和现在的新插件，使用同样的 cppcheck 选项。

结论：找出的问题完全一致。

### 总结

所有的核心关注点满足预期，可以使用 kubernetes 进行后续的开发。

## 五、可靠性保证

当集群中的节点增多后，有节点出现问题的可能性逐渐升高。我们需要确保单个节点出现问题后，正在执行的分析任务不会出现问题。

最终通过添加 processing queue 实现。正在分析的任务先放到 processing queue，如果对应的 worker pod 出现问题，将任务放回 work queue。重新由其他 worker 分析。

## 六、调度的优化

原始的设计中，存在这么两个问题：

1. 调度只是简单的一系列 kubernetes Job 串行执行。集群中一次只能存在其中一个 Job，机器资源利用率在集群变大时，会变低。
2. 没有解决多个分析任务同时出现时的竞争问题，所以无法开放“让用户手动触发分析”。

最终替换掉 kubernetes Job，改为使用 argo workflows 的 dag 解决上面的问题。

## 七、其他

有些项目是在 windows 下开发的，文件名大小写上可能没那么注意，导致实际文件的文件名和被引用时大小写不一致。可以通过构建一个对文件名大小写不敏感（开启 casefold）的 ext4文件系统，再用它做为代码存储解决，当被 nfs 引用时，它也会是对文件名大小写不敏感。

有些项目依赖的静态代码分析工具可能是基于 windows 的，这时可选的处理方案有两种：

* 制作 wine 版的 linux 容器镜像
* 提供 windows 节点制作 windows 容器镜像

## 七、最终实现出的效果

### 目标1 分析速度能够极大提升

分析耗时仅为原有的 5-7%。

| 项目 | 分析耗时 | 分布式后 | 仅有原有的 |
| ---- | -------- | -------- | --------- |
| project1 cppcheck v1.90 | 21小时18分钟[^5] | 1小时30分钟 | 7% |
| project2 cppcheck v2.3  | 100小时25分钟（近5天）[^6] | 5小时18分钟 | 5.2% |

!!! info
    “分析耗时”：使用“1台 i7-10700K”进行测试。
    “分布式后”：使用“26台 i7-10700K 组成的集群”进行测试。

### 目标2 方便查看检查出的问题

通过二次开发 codechecker 得到实现。

### 目标3 接入新项目应该容易

由平台人员编写 CRD CodeAnalysisJob 进行接入，分析配置可先放在 CRD CodeAnalysisJob 中。提供一个对外 GUI，每天下班后定时分析，也支持用户手动点击分析。对用户的项目仓库，没有任何修改。

当用户希望能自定义分析配置时，把分析配置移动到配置文件  .codeanalysis/config。

从用户的角度看，他们只是告知要接入自己项目，就可以得到一个支持定时/手动分析自己代码的服务。

### 目标4 svn, git, p4 应得到支持

利用 shell + 版本控制工具制作一个容器镜像，提供这个支持。

### 目标5 易于接入新的静态代码分析工具

只要遵循上面的《分析插件规格说明》即可，使用任何编程语言、运行环境均可。

### 目标6 静态代码分析工具插件需易于升级

上面的 CRD CodeAnalysisJob 展示了同时存在两个版本 cppcheck 插件的能力。
只需修改 CRD CodeAnalysisJob 中的 channel 即可完成升级/回滚。
