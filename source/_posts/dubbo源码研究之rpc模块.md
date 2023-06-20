---
title: dubbo源码研究之rpc模块
date: 2023-06-20 12:21:47
tags:
- dubbo
---
# dubbo源码研究之rpc模块
dubbo作为一个服务化框架，rpc模块是dubbo整个框架的核心部分。我们来通过dubbo来了解rpc调用的本质。

{% asset_img 1.png  image %}

dubbo的rpc模块以Invocation和Result为中心，扩展接口为Protocol、Invoker和Exporter。
Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。
Invoker是实体域，它是Dubbo的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起invoke调用，
它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
invoker接口提供两个方法。
Class getInterface(); 获取调用的接口
Result invoke(Invocation invocation) throws RpcException; 执行调用
说明rpc就两件事，第一，定位要调用的是哪个接口，第二发起调用返回结果。值得注意的是invoker需要Invocation提供的信息，Invocation 支持有本地invoker。invoker不参与Invocation的序列化。Invocation转成request对象通过网络传递并序列化和反序列化。

{% asset_img 2.png  image %}

rpccontext是一个静态工具类，通过threadlocal来保存单次调用的上下文。
Protocol负载invoker的暴露与引用。

{% asset_img 3.png  image %}

一个refer的invoker就会存在一个export的invoker，如果引用的服务不存在，则需要设置
暴露服务顺序

{% asset_img 4.png  image %}

引用服务时序

{% asset_img 5.png  image %}

dubbo的rpc模块也遵守dubbo的整体设计原则采用Microkernel + Plugin模式，Microkernel只负责组装Plugin。
invoker是核心模型，服务方把invoker对象暴露成Exporter对象，并把url描述写入注册中心，注册中心推送到消费方。消费方通过url转成invoker对象。





