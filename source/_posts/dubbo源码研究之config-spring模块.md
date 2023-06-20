---
title: dubbo源码研究之config-spring模块
date: 2023-06-20 13:58:06
tags:
 - dubbo
---
# dubbo源码研究之config-spring模块

dubbo-config-spring模块是dubbo-config的Extension。

{% asset_img 1.png  image %}

Dubbo的扩展点加载从JDK标准的SPI(Service Provider Interface)扩展点发现机制加强而来。
Dubbo改进了JDK标准的SPI的以下问题：
JDK标准的SPI会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过getName();获取脚本类型的名称，
但如果RubyScriptEngine因为所依赖的jruby.jar不存在，导致RubyScriptEngine类加载失败，这个失败原因被吃掉了，和ruby对应不起来，
当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。
增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。
dubbo的spi约定：在扩展类的jar包内，放置扩展点配置文件：META-INF/dubbo/接口全限定名，内容为：配置名=扩展实现类全限定名，多个实现类用换行符分隔。

{% codeblock schema.xml lang:java   %}
    
    @SPI
    public interface ExtensionFactory {

    /** 
     * Get extension. 
     *  
     * @param type object type. 
     * @param name object name. 
     * @return object instance. 
     */  
    <T> T getExtension(Class<T> type, String name);  

    }
{% endcodeblock %}
dubbo的扩展点接口ExtensionFactory ，该接口有三个实现

{% asset_img 2.png  image %}

dubbo的扩展实现的具体类是ExtensionLoader。
ExtensionLoader加载扩展点时，会检查扩展点的属性（通过set方法判断），如该属性是扩展点类型，则会注入扩展点对象。
因为注入时不能确定使用哪个扩展点（在使用时确定），所以注入的是一个自适应扩展（一个代理）。自适应扩展点调用时，选取一个真正的扩展点，并代理到其上完成调用。
Dubbo是根据调用方法参数（上面有调用哪个扩展点的信息）来选取一个真正的扩展点。


{% codeblock  lang:java   %}

    @SuppressWarnings("unchecked")  
    private T createExtension(String name) {  
    Class<?> clazz = getExtensionClasses().get(name);  
        if (clazz == null) {  
            throw findException(name);  
        }  
        try {  
            T instance = (T) EXTENSION_INSTANCES.get(clazz);  
            if (instance == null) {  
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());  
                instance = (T) EXTENSION_INSTANCES.get(clazz);  
            }  
            injectExtension(instance);  
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;  
    if (wrapperClasses != null && wrapperClasses.size() > 0) {  
    for (Class<?> wrapperClass : wrapperClasses) {  
    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));  
    }  
    }  
    return instance;  
    } catch (Throwable t) {  
    throw new IllegalStateException("Extension instance(name: " + name + ", class: " +  
    type + ")  could not be instantiated: " + t.getMessage(), t);  
    }  
    }

    private T injectExtension(T instance) {  
    try {  
    if (objectFactory != null) {  
    for (Method method : instance.getClass().getMethods()) {  
    if (method.getName().startsWith("set")  
    && method.getParameterTypes().length == 1  
    && Modifier.isPublic(method.getModifiers())) {  
    Class<?> pt = method.getParameterTypes()[0];  
    try {  
    String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";  
    Object object = objectFactory.getExtension(pt, property);  
    if (object != null) {  
    method.invoke(instance, object);  
    }  
    } catch (Exception e) {  
    logger.error("fail to inject via method " + method.getName()  
    + " of interface " + type.getName() + ": " + e.getMessage(), e);  
    }  
    }  
    }  
    }  
    } catch (Exception e) {  
    logger.error(e.getMessage(), e);  
    }  
    return instance;  
    }
{% endcodeblock %}

可以看出ExtensionLoader持有某一类扩展点的所有扩展，并且扩展以class为key，一个扩展类只有一个实例。
如果扩展点使用了Adaptive则会通过字节码生成一个自适应的扩展类。
通过createAdaptiveExtensionClassCode方法返回class的code，然后通过Compiler接口返回class对象

{% codeblock  lang:java   %}
    
    /**
    * Compiler. (SPI, Singleton, ThreadSafe)
    * 
    * @author william.liangf
    */  
    @SPI("javassist")  
    public interface Compiler {

    /**
      * Compile java source code.
      *
      * @param code Java source code
      * @param classLoader TODO
      * @return Compiled class
        */  
        Class<?> compile(String code, ClassLoader classLoader);

      }

{% endcodeblock %}

Compiler dubbo支持jdk和Javassist两种实现，特别注意一点，JdkCompiler的java版本通过硬编码指定版本为1.6。
JdkCompiler首先通过JavaFileObject接口生成java文件，然后通过JavaCompiler接口编译成class文件，最后通过ClassLoader加载生成的class文件。
spring名称空间扩展
dubbo通过扩展spring的名称空间来读取xml配置。
dubbo.xsd是spring schma的扩展文件。作用是定义相关xml元素和名称空间。
spring.handlers,spring.schemas是spring在解析xml的时候根据spi机制读取对象的NamespaceHandler和xsd文件位置。
很多程序猿使用dubbo开发的时候，会发现ide工具报错，因为xml上面dubbo名称空间的链接打不开，其实这个报错可以无视的，
因为spring解析的xml的时候这个网络地址其实是指向spi配置文件里面的一个java类。
spring.handlers

{% codeblock  lang:xml   %}

    http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
{% endcodeblock %}

DubboNamespaceHandler则注册了相关自定义元素的解析器

{% codeblock  lang:xml   %}

    /**
     * DubboNamespaceHandler
     * 
     * @author william.liangf
     * @export
    */  
    public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {  
    Version.checkDuplicate(DubboNamespaceHandler.class);  
    }

    public void init() {  
    registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));  
    registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));  
    registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));  
    registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));  
    registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));  
    registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));  
    registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));  
    registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));  
    registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));  
    registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));  
    }

    }
{% endcodeblock %}


