---
title: kubernetes简单了解
date: 2018-12-12 00:08:35
tags:
    - Micro Service
---

## 什么是kubernetes
一个可以自动部署、拓展和管理容器化的应用（比如运行在Docker容器里面的应用）的系统，开源的。英文缩写为k8s。

## 它可以做什么
它可以让你快速地部署分布式应用，让你在发布和升级应用的时候，保持应用正常运转，不需要停止应用。它是一个容器化应用编排器，能够帮你把各个容器化应用编排好，让它们能够有条不紊地运作。比如你在部署了一个应用之后，要进行版本升级，k8s会创建新的pod来试探性地部署新版本的应用，确保新版本的应用能正常运作之后再把旧版本的引用一个个地替换成新版本的应用，因此如果新版本的应用部署失败了，也完全不会影响就版本的应用。

### 服务注册与发现
当你用k8s部署多个需要相互协作的应用的时候，k8s就相当于一个注册中心（如zookeeper, consul, eureka）。使用k8s部署应用，你的应用不再需要一个相应的客户端来注册和发现服务，k8s的pod和service已经帮你实现了这些东西，除非你需要客户端的负载均衡（这里指的是客户端需要明确指定每次需要访问哪台服务器的那种负载均衡，这需要获取服务器的信息，所以要在代码层面显式地和k8s进行通信获取信息），这时候你才需要在应用的代码里面和k8s进行通信来获取负载均衡需要的信息。这点非常不错，因为如过我们使用eureka实现服务注册与发现的话，那我们就需要一个相应的客户端来和eureka服务端来进行通信，然后向zookeeper注册服务，获取服务等等。问题是，eureka是用Java实现的，它默认只有Java客户端，如果你用的其他语言，你需要自己实现一个客户端和eureka服务端进行通信。

__Pod的图解__
![](https://d33wubrfki0l68.cloudfront.net/fe03f68d8ede9815184852ca2a4fd30325e5d15a/98064/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)

一个Pod里面可以运行一个容器化应用（这个应用里面可能跑多个程序，比如一个web后端程序和一个数据库），Pod是有生命周期的，随时可能应为应用故障被移除停止或者部署新应用而被创建，Pod在Node里面运行，一个Node能放任意个Pod。

__Node的图解__
![](https://d33wubrfki0l68.cloudfront.net/5cb72d407cbe2755e581b6de757e0d81760d5b86/a9df9/docs/tutorials/kubernetes-basics/public/images/module_03_nodes.svg)

Node对应一台物理主机或者虚拟机，能放任意个Pod，一个Node里面至少运行着一个kubelet进程和容器程序（比如Docker)

__Service的图解__
![](https://d33wubrfki0l68.cloudfront.net/cc38b0f3c0fd94e66495e3a4198f2096cdecd3d5/ace10/docs/tutorials/kubernetes-basics/public/images/module_04_services.svg)

Service以一个或者多个Pod为基本逻辑单位，图上面画得很清楚了，更多细节可以查看[官方文档](https://kubernetes.io/docs/tutorials/kubernetes-basics/)


### 负载均衡
k8s的负载均衡是服务端的负载均衡，也就是当用户访问某个服务的时候，用户的请求会根据负载均衡算法被分配到应用集群的其中一个应用上面（其实是某一个pod上面），这得益于k8s的Service抽象，在k8s里面，一个应用部署之后，是运行在一个pod里面的，假设这个应用一共部署了5个实例，将会创建5个pod来运行这些应用，组成一个集群，每个pod都有相应的ip地址，但是这些ip地址是内部的ip地址，外部不能直接访问到（在同一个k8s集群的网络里面的应用是可以相互访问的），需要通过Service暴露给外部, 当用户访问这些应用提供的服务的时候，会由它们对应的Service通过Label和Selector定位到这些应用，然后决定将请求交给哪一个应用处理。那么Service如何定位呢？Service可以通过[不同的配置方式](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#exposing-the-service)来进行定位

### 健康检查
k8s能自动重启启动失败的应用，当一个Node挂掉后，它会使用其他可用的Node重新部署挂掉的Node里面的应用，当一个应用没有响应你自定义的健康检查的时候，它会停掉这个应用，并且不把这个应用暴露给客户端，直到这个应用恢复正常。

##简单上手
要简单地上手，最快的方式其实是用官网上的tutorial里面的网页终端，但是我这里网络状况比较糟糕，只能本地安装了
###安装kubectl
我的机器系统是Ubuntu18.10，按照文档输入几个命令就行了。
//TODO
