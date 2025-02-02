---
type: docs
title: "店小蜜升级 Triple 协议"
linkTitle: "达摩院云小蜜"
weight: 4
---

# 前言
阿里云-达摩院-云小蜜对话机器人产品基于深度机器学习技术、自然语言理解技术和对话管理技术，为企业提供多引擎、多渠道、多模态的对话机器人服务。17年云小蜜对话机器人在公共云开始公测，同期在混合云场景也不断拓展。为了同时保证公共云、混合云发版效率和稳定性，权衡再三我们采用了1-2个月一个大版本迭代。
经过几年发展，为了更好支撑业务发展，架构升级、重构总是一个绕不过去的坎，为了保证稳定性每次公共云发版研发同学都要做两件事：

 	1. 梳理各个模块相较线上版本接口依赖变化情况，决定十几个应用的上线顺序、每批次发布比例；
 	2. 模拟演练上述1产出的发布顺序，保证后端服务平滑升级，客户无感知；

上述 1、2 动作每次都需要 2-3 周左右的时间梳理、集中演练，但是也只能保证开放的PaaS API平滑更新；

控制台服务因为需要前端、API、后端保持版本一致才能做到体验无损（如果每次迭代统一升级API版本开发、协同成本又会非常大），权衡之下之前都是流量低谷期上线，尽量缩短发布时间，避免部分控制台模块偶发报错带来业务问题。针对上面问题，很早之前就考虑过用蓝绿发布、灰度等手段解决，但是无奈之前对话机器人在阿里云内部业务区域，该不再允许普通云产品扩容，没有冗余的机器，流量治理完全没法做。

# 迁移阿里云云上

带着上面的问题，终于迎来的 2021 年 9 月份，云小蜜将业务迁移至阿里云云上。

## Dubbo3 的实践

“当时印象最深的就是这张图，虽然当时不知道中间件团队具体要做什么事情，但是记住了两个关键词：三位一体、红利。没想到在2021年底，真真切切享受到了这个红利。”

![image1](/imgs/v3/users/yunxiaomi-1.png)

云小蜜使用的是集团内部的HSF服务框架，需要迁移至阿里云云上，并且存在阿里云云上与阿里内部业务域的互通、互相治理的诉求。云小蜜的公共服务部署在公有云VPC，部分依赖的数据服务部署在内部，内部与云上服务存在RPC互调的诉求，其实属于混合云的典型场景。
简单整理了下他们的核心诉求，概括起来有以下三点吧：希望尽可能采用开源方案，方便后续业务推广；在网络通信层面需要保障安全性；对于业务升级改造来说需要做到低成本。

![image2](/imgs/v3/users/yunxiaomi-2.png)

在此场景下，经过许多讨论与探索，方案也敲定了下来

- 全链路升级至开源 Dubbo3.0，云原生网关默认支持Dubbo3.0，实现透明转发，网关转发RT小于1ms
- 利用 Dubbo3.0 支持HTTP2特性，云原生网关之间采用 mTLS 保障安全
- 利用云原生网关默认支持多种注册中心的能力，实现跨域服务发现对用户透明，业务侧无需做任何额外改动
- 业务侧升级SDK到支持 Dubbo3.0，配置发布 Triple 服务即可，无需额外改动

**解决了互通、服务注册发现的问题之后，就是开始看如何进行服务治理方案了**

# 阿里云云上流量治理

迁移至阿里云云上之后，流量控制方案有非常多，比如集团内部的全链路方案、集团内单元化方案等等。

## 设计目标和原则

1. 要引入一套流量隔离方案，上线过程中，新、旧两个版本服务同时存在时，流量能保证在同一个版本的“集群”里流转，这样就能解决重构带来的内部接口不兼容问题。
2. 要解决上线过程中控制台的平滑性问题，避免前端、后端、API更新不一致带来的问题。
3. 无上线需求的应用，可以不参与上线。
4. 资源消耗要尽量少，毕竟做产品最终还是要考虑成本和利润。

## 方案选型

1. 集团内部的全链路方案：目前不支持阿里云云上
2. 集团内单元化方案：主要解决业务规模、容灾等问题，和我们碰到的问题不一样
3. 搭建独立集群，版本迭代时切集群：成本太大
4. 自建：在同一个集群隔离新、老服务，保证同一个用户的流量只在同版本服务内流转

以RPC为例：

* 方案一：通过开发保证，当接口不兼容升级时，强制要求升级HSF version，并行提供两个版本的服务； 缺点是一个服务变更，关联使用方都要变更，协同成本特别大，并且为了保持平滑，新老接口要同时提供服务，维护成本也比较高
* 方案二：给服务（机器）按版本打标，通过RPC框架的路由规则，保证流量优先在同版本内流转

## 全链路灰度方案

就当1、2、3、4都觉得不完美，一边调研一边准备自建方案5的时候，兜兜绕绕拿到了阿里云 MSE 微服务治理团队[《20分钟获得同款企业级全链路灰度能力》](https://yuque.antfin.com/docs/share/a8df43ac-3a3b-4af4-a443-472828884a5d?#)，方案中思路和准备自建的思路完全一致，也是利用了RPC框架的路由策略实现的流量治理，并且实现了产品化（微服务引擎-微服务治理中心），同时，聊了两次后得到几个“支持”，以及几个“后续可以支持”后，好像很多事情变得简单了...

![image3](/imgs/v3/users/yunxiaomi-3.png)

从上图可以看到，各个应用均需要搭建基线(base)环境和灰度(gray)环境，除了流量入口-业务网关以外，下游各个业务模块按需部署灰度（gray）环境，如果某次上线某模块没有变更则无需部署。

### 各个中间件的治理方案

1. Mysql、ElasticSearch：持久化或半持久化数据，由业务模块自身保证数据结构兼容升级；
2. Redis：由于对话产品会有多轮问答场景，问答上下文是在Redis里，如果不兼容则上线会导致会话过程中的C端用户受影响，因此目前Redis由业务模块自身保证数据结构兼容升级；
3. 配置中心：基线(base)环境、灰度(gray)环境维护两套全量配置会带来较大工作量，为了避免人工保证数据一致性成本，基线(base)环境监听dataId，灰度(gray)环境监听gray.dataId如果未配置gray.dataId则自动监听dataId；（云小蜜因为在18年做混合云项目为了保证混合云、公共云使用一套业务代码，建立了中间件适配层，本能力是在适配层实现）
4. RPC服务：使用阿里云 one agent 基于Java Agent技术利用Dubbo框架的路由规则实现，无需修改业务代码；

应用只需要加一点配置：

* 1）linux环境变量
alicloud.service.tag=gray标识灰度，基线无需打标
profiler.micro.service.tag.trace.enable=true标识经过该机器的流量，如果没有标签则自动染上和机器相同的标签，并向后透传
* 2）JVM参数，标识开启MSE微服务流量治理能力**       SERVICE_OPTS=**"$**{SERVICE_OPTS}** -Dmse.enable=true"**

### 流量管理方案

流量的分发模块决定流量治理的粒度和管理的灵活程度。

对话机器人产品需要灰度发布、蓝绿发布目前分别用下面两种方案实现：

1. 灰度发布：
部分应用单独更新，使用POP的灰度引流机制，该机制支持按百分比、UID的策略引流到灰度环境
2. 蓝绿发布：
    * 1）部署灰度(gray)集群并测试：测试账号流量在灰度(gray)集群，其他账号流量在基线(base)集群
    * 2）线上版本更新：所有账号流量在灰度(gray)集群
    * 3）部署基线(base)集群并测试：测试账号流量在基线(base)集群，其他账号流量在灰度(gray)集群
    * 4）流量回切到基线(base)集群并缩容灰度(gray)环境：所有账号流量在基线(base)集群

## 全链路落地效果

上线后第一次发布的效果：“目前各个模块新版本的代码已经完成上线，含发布、功能回归一共大约花费2.5小时，相较之前每次上线到凌晨有很大提升。”
MSE 微服务治理全链路灰度方案满足了云小蜜业务在高速发展情况下快速迭代和小心验证的诉求，通过JavaAgent技术帮助云小蜜快速落地了企业级全链路灰度能力。
流量治理随着业务发展会有更多的需求，下一步，我们也会不断和微服务治理产品团队，扩充此解决方案的能力和使用场景，比如：rocketmq、schedulerx的灰度治理能力。

## 更多的微服务治理能力

使用 MSE 服务治理后，发现还有更多开箱即用的治理能力，能够大大提升研发的效率。包含服务查询、服务契约、服务测试等等。这里要特别提一下就是云上的服务测试，服务测试即为用户提供一个云上私网 Postman ，让我们这边能够轻松调用自己的服务。我们可以忽略感知云上复杂的网络拓扑结构，无需关系服务的协议，无需自建测试工具，只需要通过控制台即可实现服务调用。支持 Dubbo 3.0 框架，以及 Dubbo 3.0 主流的 Triple 协议。

![image4](/imgs/v3/users/yunxiaomi-4.png)

# 结束语

最终云小蜜对话机器人团队成功落地了全链路灰度功能，解决了困扰团队许久的发布效率问题。在这个过程中我们做了将部分业务迁移至阿里云云上、服务框架升级至Dubbo3.0、选择MSE微服务治理能力等等一次次新的选择与尝试。“世上本没有路，走的人多了便成了路”。经过我们工程师一次又一次的探索与实践，能够为更多的同学沉淀出一个个最佳实践。我相信这些最佳实践将会如大海中璀璨的明珠般，经过生产实践与时间的打磨将会变得更加熠熠生辉。


