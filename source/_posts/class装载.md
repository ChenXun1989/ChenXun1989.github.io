---
title: class装载
date: 2023-06-20 15:18:35
tags:
 - java源码
---
# class装载

**示例代码链接 https://github.com/ChenXun1989/study-example**

演示demo

{% codeblock  lang:java   %}

    public class TestClassLoader {
    public static void main(String[] args) {
    ClassLoader myClassLoader = new ClassLoader() {};
    try {
    Class cls = myClassLoader.loadClass(TestClassLoader.class.getName());
    System.out.println(myClassLoader.getParent());
    System.out.println(cls.getClassLoader());
    } catch (ClassNotFoundException e) {
    e.printStackTrace();
    }
    }
    }


{% endcodeblock %}

输出结果：

{% codeblock  lang:java   %}

    sun.misc.Launcher$AppClassLoader@18b4aac2
    sun.misc.Launcher$AppClassLoader@18b4aac2
{% endcodeblock %}

从结果上我们可以得出以下两个结论
1. 自定义的classloader的parent默认为AppClassLoader
2. 当classloader加载某个class的时候会委托给parent（父加载器）加载

## 双亲委托细节
双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，他首先不会自己去尝试加载这个类，
而是把这个请求委派父类加载器去完成。每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，
只有当父加载器反馈自己无法完成这个请求（他的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

双亲委托是如何实现的呢？秘密就在loadClass 这个方法

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，他首先不会自己去尝试加载这个类，
而是把这个请求委派父类加载器去完成。每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，
只有当父加载器反馈自己无法完成这个请求（他的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

双亲委托是如何实现的呢？秘密就在loadClass 这个方法

{% codeblock  lang:java   %}

    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
    if (c == null) {
    long t0 = System.nanoTime();
    try {
    if (parent != null) {
    c = parent.loadClass(name, false);
    } else {
    c = findBootstrapClassOrNull(name);
    }
    } catch (ClassNotFoundException e) {
    // ClassNotFoundException thrown if class not found
    // from the non-null parent class loader
    }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);
      
                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
    }


{% endcodeblock %}

从code上得出结论如果没有知道class,会先通过父加载起去加载，一直递归上去，如果没有父加载器直接通过bootstrap加载器加载
大致流程如下： 用户自定义的classlaoder – AppClassLoader —ExtClassLoader – bootStrapClassLoader

{% asset_img 1.png  image %}

这么做有什么好处？

>如果没有使用双亲委派模型，由各个类加载器自行加载的话，如果用户自己编写了一个称为java.lang.Object的类，
并放在程序的ClassPath中，那系统将会出现多个不同的Object类， Java类型体系中最基础的行为就无法保证。应用程序也将会变得一片混乱

## 如何破坏双亲委托
双亲委托并不是一个强制约束，只是一种大家都遵守规范，但有些场景会破坏双亲委托（例如SPI）

{% codeblock  lang:java   %}

    public static void  test2(){  
    ServiceLoader<SPITest> loaders= ServiceLoader.load(SPITest.class);
    loaders.forEach(i->{
    i.test();
    });
    }
{% endcodeblock %}

准确的说spi没有破坏双亲委托，只是绕过了双亲委托获取class，委托机制上层（例如bootstrapClassloader）通过Thread.currentThread().getContextClassLoader()来获取
委托机制里面下层（例如AppClassLoader）加载的类。具体code如下：

{% codeblock  lang:java   %}

    public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
    }

{% endcodeblock %}

这样做是为了解决一些 上层类依赖下层类的情况，比如jdbc，jdbc相关的类文件在rt.jar中，但是driver对应的实现类是放在我们应用的jar的lib里面，
这样 extClassLoader 就加载不到，但是通过spi机制，rt.jar包中的jdbc相关类就能获取由AppClassLoader 加载的jdbc具体的driver了。
上面讲到 双亲委托机制是通过loaderclass这个方法来实现的，如果我们自定义的classloader覆盖改方法（或者覆盖findClass），就能强行破坏，例如osgi.
在jvm里面每个class的唯一标识符由 classloader全名+class全名构成，因此假设两个独立的classloader，加载一样的class，jvm也会认为是两个独立的class，
tomcat就是此机制来实现webapp目录下不同的web应用的隔离

