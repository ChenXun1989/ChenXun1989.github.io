---
title: RPC框架设计
tags:
  - 架构设计
abbrlink: 59976
date: 2023-06-20 15:08:31
---
# RPC框架设计
RPC（Remote Procedure Call Protocol）——远程过程调用协议，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。
RPC协议假定某些传输协议的存在，如TCP或UDP，为通信程序之间携带信息数据。在OSI网络通信模型中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加容易。
**代码地址：https://github.com/ChenXun1989/netty-rpc**

## 透明化RPC
一个RPC服务调用有以下几个步骤
1. 服务消费方（client）调用以本地调用方式调用服务
2. client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体
3. client stub找到服务地址，并将消息发送到服务端
4. server stub收到消息后进行解码
5. server stub根据解码结果调用本地的服务
6. 本地服务执行并将结果返回给server stub
7. server stub将返回结果打包成消息并发送至消费方
8. client stub接收到消息，并进行解码
9. 服务消费方得到最终结果

**透明的RPC测试把2-8步骤全部封装起来，消费方只需要执行第一个步骤，并等待返回结果就行。**

透明化RPC具体调用过程
{% asset_img 1.png  image %}
## 注意要点
- 动态代理（客户端） 主要实现有两种JDK动态代理和cglib
- reqeust对象 主要包含 服务方法签名和参数
- repsone对象 主要包含服务方法处理结果和异常
- 序列化和反序列化 主要在 reqeust和respone对象，该demo采用protobuf实现
- socket通讯 该demo采用netty4实现。

