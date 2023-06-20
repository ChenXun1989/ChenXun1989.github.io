---
title: mybatis源码分析（一）
date: 2023-06-20 14:59:40
tags:
- mybatis
---
# mybatis源码分析（一）

## sqlSessionFactory对象
sqlSessionFactory 是mybatis 核心配置类，管理mybatis全局配置。

## sqlSession接口
一个或者多个sql操作的执行单元。实现一个完整的sql操作。依赖sqlSessionFactory创建

## Executor接口
对应jdbc底层一个完整的sql操作。sqlSession 实际通过Executor执行sql操作

## MapperProxy类
代理实现mybatis的客户端mapper接口

## MapperMethod类
对应客户端代码的mapper接口里面的一个方法。该实例缓存了。

## StatementHandler接口
jdbc Statement的装饰器

## ResultSetHandler接口
jdbc resultSet的装饰器

## mybaits执行orm操作细节
{% asset_img 1.png  image %}
**sqlsession的close 方法会关闭底层jdbc connection**

## mybatis插件机制
mybatis插件是基于代理实现的，具体支持一下四个接口

- Executor
- ParameterHandler
- ResultSetHandler
- StatemetHandler



