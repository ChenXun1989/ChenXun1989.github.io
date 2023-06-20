---
title: dubbo源码研究之extension模块
date: 2023-06-19 17:20:53
tags:
- dubbo
---
# dubbo源码研究之extension模块

dubbo的扩展采用spi机制实现，
spi（Service Provider Interface）是指一些提供给你继承、扩展，完成自定义功能的类、接口或者方法。
spi把控制权利交个调用方，调用方来决定使用该spi的哪个实现。
dubbo扩展机制的核心类是ExtensionLoader，该类通过静态方法getExtensionLoader获取一个指定接口的ExtensionLoader实例。

{% codeblock ExtensionLoader.java  lang:java  %}

    @SuppressWarnings("unchecked")  
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {  
    if (type == null)  
    throw new IllegalArgumentException("Extension type == null");  
    if(!type.isInterface()) {  
    throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");  
    }  
    if(!withExtensionAnnotation(type)) {  
    throw new IllegalArgumentException("Extension type(" + type +   
    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");  
    }
    
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);  
    if (loader == null) {  
    EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));  
    loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);  
    }  
    return loader;  
    }
{% endcodeblock %}

该方法要求通过spi实现的接口上必须包含@spi注解，并且一个接口的ExtensionLoader是唯一的，保存在静态容器EXTENSION_LOADERS（ConcurrentHashMap）中。
ExtensionLoader提供实例方法getExtension获取该接口的具体实现。

{% codeblock ExtensionLoader.java  lang:java  %}

    public T getExtension(String name) {  
    if (name == null || name.length() == 0)  
    throw new IllegalArgumentException("Extension name == null");  
    if ("true".equals(name)) {  
    return getDefaultExtension();  
    }  
    Holder<Object> holder = cachedInstances.get(name);  
    if (holder == null) {  
    cachedInstances.putIfAbsent(name, new Holder<Object>());  
    holder = cachedInstances.get(name);  
    }  
    Object instance = holder.get();  
    if (instance == null) {  
    synchronized (holder) {  
    instance = holder.get();  
    if (instance == null) {  
    instance = createExtension(name);  
    holder.set(instance);  
    }  
    }  
    }  
    return (T) instance;  
    }
{% endcodeblock %}

具体流程如下

{% asset_img 1.png  image %}

该方法通过大量缓存容器来优化性能，并且每个扩展点都是单例存在，所以扩展dubbo框架的时候要注意该扩展点的线程安全性。
ExtensionFactory为spi接口实现实例在注入属性（injectExtension）时提供注入的属性.

{% asset_img 2.png  image %}

该工厂有三个实现，分别支持从spring ，spi，Adaptive里面获取对象，注入Extension对象中。

