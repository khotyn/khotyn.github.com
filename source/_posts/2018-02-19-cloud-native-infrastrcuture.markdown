---
layout: post
title: "Cloud Native Infrastrcuture 阅读笔记"
date: 2018-02-19 21:00
comments: true
categories: 
---

这是假期读的第二本书，以下是对本书的阅读的内容的一些总结：

就 Cloud native 这个词来说，已经被市场人员到处在用了，就跟 Microservice 一样，很难清晰地去定义到底什么是 Cloud native。在本书中，作者提到的 Cloud native 其实包含了两个部分的内容，一个是 Cloud native 的基础设施（Cloud native infrastructure），另一个是 Cloud native 的应用（Cloud native application）。作者对于 Cloud native infrastructure 的定义是：

> Cloud native infrastructure is infrastructure that is hidden behind useful abstractions, controlled by APIs, managed by software, and has the purpose of running applications. Running infrastructure with these traits gives rise to a new pattern for managing that infrastructure in a scalable, efficient way.  

所谓的 Cloud native 的基础设施就是当我们的基础设施（大量的 VM/Docker，复杂的网络，各种各样的储存等等）变的非常复杂和庞大的时候，通过人力是无法高效地进行运维的，在这样的大规模下，我们应该尽量地去避免人工介入，而是需要有一套软件来帮助我们更好地去管理我们的基础设施，可以让运行在这套基础设施上 Cloud  native 的应用可以非常方便地被运维，部署，监控等等。那么怎么去设计管理 Cloud native 的基础设施的系统呢，作者提到了几个点，其中包括系统的自举，API 的设计等等，但是我觉得其中最重要的一点是 Reconciler Pattern，关于这个 Pattern，书中提到了四个原则：

* Use a data structure for all inputs and outputs.
* Ensure that the data structure is immutable.
* Keep the resource map simple.
* Make the actual state match the expected state.

其中最后一点和 Reconciler Pattern 的关系最大，这个也是 Kubernetes 中的方式，我们往 Kubernetes 中去提交一个 Spec，比如要求一个应用的实例的数量应该是 4，那么 Kubernetes 就会尽可能地去保证这个应用的实例的数量是 4，如果一个实例 Crash 了，它就马上新起一个实例。这就是所谓的 Reconciler Pattern，尽量让实际的状态可以和期望的状态匹配上。

Reconciler Pattern 在设计上非常大的一个优势就是它是声明式的，而不是反应式的。用户要做的是按照系统提供出的 API，直接告诉系统你想要什么，比如说，我想要一个应用的实例的数量保持在 4 个，系统就去考虑各种情况，让你的应用的数量保持在 4 个。而反应式的话，则是监听系统的事件，比如你的应用的某个实例 Crash 了，然后你监听到了这个事件之后，就调用系统的 API 去新创建一个实例。显然，对于用户来说，声明式的方式要简单地很多，不容易出错。而反应式则要求每一个应用都去监听系统的事件，不但侵入到了应用，也很容易出错。

在上面的 Reconciler Pattern 的四个原则中，第二点让数据结构是不可变的这一点也是非常重要的，但是我不是很明白这个点和 Reconciler Pattern 具体的关系在哪里？这个原则是说如果我们要修改一个数据结构，我们实际上不是修改它，而是新建一个新的，然后把原来的标记成过期，这样的好处是，我们可以保留数据结构中间的各种版本，当需要看下当前的 Infra 有哪些区别的，就可以把这些版本直接拿出来对比，非常方便地就可以回答线上环境到底发生了什么样的变更。

有了 Cloud native 的基础设施，我们还要有 Cloud native 的应用运行在上面，那么作为基础设施，我们还应该给应用到底提供什么样的能力呢，书中提到了八点：

* Runtime and isolation（这里的 isolation 是指资源的 isolation，比如 CPU，Memory，Storage 等等）
* Resource allocation and scheduling
* Environment isolation（这里的 isolation 指的是环境的 isolation，比如 dev，test，staging，product）
* Service discovery
* State management（Readiness, Liveness）
* Monitoring and logging
* Metrics aggregation
* Debugging and tracing

我们可以看到，上面的八个点中其实下面非常多的都是原来传统的中间件在干的一些事情，而在 Cloud native 的基础设施中，这些中间件正在往下沉，直接成为基础设施的一部分，为应用提供能力。而且在 CNCF 中，上面的八个点基本上都有一个对应的产品对应：

* Runtime and isolation: containers, rkt
* Resource allocation and scheduling: kubernetes
* Environment isolation: kubernetes
* Service discovery: kubernetes
* State management: kubernetes
* Monitoring and logging: Prometheus, Fluentd
* Metrics aggregation: Prometheus
* Debugging and tracing: OpenTracing, Jaeger

除了上面的八个点之外，我个人认为应该还加上一个点，就是 Network Resilient，这可以极大程度地解决应用之间的网络通信的问题，这个正是 Service Mesh 所提供的能力，在 CNCF 里面对应的产品是 Envoy（奇怪 istio 怎么还没有进入 CNCF）。

总结来说，本书的作者基本上把设计一个 Cloud native infrastructure 所需要做的事情都讲了一遍，包括上面没有提到的测试等等，大部分的内容其实就是 Kubernetes 目前已经做到的一些事情，如果你对 Kubernetes 的设计已经非常清楚了，那么这本书对你的价值可能不大，如果对 Kubernetes 的设计并不熟悉，那么相信你可以从这本书里面学到不少东西。