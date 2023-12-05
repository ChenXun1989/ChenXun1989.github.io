---
title: dubbo源码研究之container模块
tags:
  - dubbo
abbrlink: 172
date: 2023-06-20 12:25:41
---
# dubbo源码研究之container模块
dubbo-container模块是dubbo启动顺序中的第一个模块，
dubbo-container模块是容器模块，通过dubbo-container模块读取dobbo-config模块的相关配置。

{% codeblock lang:java   %}

    /**
    * Container. (SPI, Singleton, ThreadSafe)
    * 
    * @author william.liangf
    */  
    @SPI("spring")  
    public interface Container {

    /**
      * start.
        */  
        void start();

    /**
      * stop.
        */  
        void stop();
    }

{% endcodeblock %}

container接口非常简洁，只有两个方法，start，stop。注意两个方法都是void并且不抛出受检查异常。
细心的童鞋也可能发现上面的注释，spi，Singleton，ThreadSafe。 说明该接口是基于spi机制，单例，并且线程安全的。

Container 接口的主要实现

{% asset_img 1.png  image %}

其中JavaConfigContainer是基于spring的javaconfig，其中javaconfig是spring4之后主推的配置模式，
使用spring boot的童鞋如果和dubbo整合使用零配置的方式可以考虑这个类。
SpringContainer主要指定了默认的配置文件路径，通过ClassPathXmlApplicationContext来启动spring容器。

{% codeblock lang:java   %}
    
    /**
    * SpringContainer. (SPI, Singleton, ThreadSafe)
    * 
    * @author william.liangf  
    */  
    public class SpringContainer implements Container {

    private static final Logger logger = LoggerFactory.getLogger(SpringContainer.class);

    public static final String SPRING_CONFIG = "dubbo.spring.config";

    public static final String DEFAULT_SPRING_CONFIG = "classpath*:META-INF/spring/*.xml";

    static ClassPathXmlApplicationContext context;

    public static ClassPathXmlApplicationContext getContext() {  
    return context;  
    }

    public void start() {  
    String configPath = ConfigUtils.getProperty(SPRING_CONFIG);  
    if (configPath == null || configPath.length() == 0) {  
    configPath = DEFAULT_SPRING_CONFIG;  
    }  
    context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"));  
    context.start();  
    }

    public void stop() {  
    try {  
    if (context != null) {  
    context.stop();  
    context.close();  
    context = null;  
    }  
    } catch (Throwable e) {  
    logger.error(e.getMessage(), e);  
    }  
    }

}

{% endcodeblock %}

其他几个实现类似，重点讲下com.alibaba.dubbo.container.Main。这个是dubbo自带的main函数入口。

{% codeblock lang:java   %}

        public static void main(String[] args) {  
        try {  
        if (args == null || args.length == 0) {  
        String config = ConfigUtils.getProperty(CONTAINER_KEY, loader.getDefaultExtensionName());  
        args = Constants.COMMA_SPLIT_PATTERN.split(config);  
        }

         final List<Container> containers = new ArrayList<Container>();  
         for (int i = 0; i < args.length; i ++) {  
             containers.add(loader.getExtension(args[i]));  
         }  
         logger.info("Use container type(" + Arrays.toString(args) + ") to run dubbo serivce.");  
           
         if ("true".equals(System.getProperty(SHUTDOWN_HOOK_KEY))) {  
          Runtime.getRuntime().addShutdownHook(new Thread() {  
              public void run() {  
                  for (Container container : containers) {  
                      try {  
                          container.stop();  
                          logger.info("Dubbo " + container.getClass().getSimpleName() + " stopped!");  
                      } catch (Throwable t) {  
                          logger.error(t.getMessage(), t);  
                      }  
                      synchronized (Main.class) {  
                          running = false;  
                          Main.class.notify();  
                      }  
                  }  
              }  
          });  
         }  
           
         for (Container container : containers) {  
             container.start();  
             logger.info("Dubbo " + container.getClass().getSimpleName() + " started!");  
         }  
         System.out.println(new SimpleDateFormat("[yyyy-MM-dd HH:mm:ss]").format(new Date()) + " Dubbo service server started!");  
     } catch (RuntimeException e) {  
         e.printStackTrace();  
         logger.error(e.getMessage(), e);  
         System.exit(1);  
     }  
     synchronized (Main.class) {  
         while (running) {  
             try {  
                 Main.class.wait();  
             } catch (Throwable e) {  
             }  
         }  
     }  
    }

{% endcodeblock %}

该方法通过钩子 Runtime.getRuntime().addShutdownHook来实现优雅停机，同时还支持通过启动参数来加载扩展点，
containers.add(loader.getExtension(args[i]))。但是没有类似spring boot 的CommandLineRunner 接口，
在容器启动完毕之后执行一些初始化动作。建议使用dubbo的童鞋如没有特殊需求，可以使用该类作为程序启动类。

