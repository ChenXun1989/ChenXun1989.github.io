---
title: IOC容器设计
date: 2023-06-20 15:04:16
tags:
---
#IOC容器设计
提供getBean接口，支持依赖注入
**示例代码地址： https://github.com/ChenXun1989/ioc-framework**
##IOC容器主要模块
- ApplicationContext，IOC容器上下文，持有对象Map
- BeanFactory 每一个类型（class） 对应一BeanFactory，提供获取类实现的接口，属性注入接口
- CompentScan 收集相关配置信息，主要分xml和注解扫描两种

##IOC容器启动过程
ApplicationContext 创建
{% codeblock  lang:java   %}

    ApplicationContext context=new ApplicationContext("com.chenxun.framework.example.entity");
    Student s=(Student) context.getBean("student");
    s.test();

{% endcodeblock %}
CompentScan 收集相关配置信息，解析出被ioc容器托管的class集合

{% codeblock  lang:java   %}

    private final List<String > list=new ArrayList<>();	 
    public void scan(String packageName){
    String parent=	System.getProperty("user.dir");
    parent=parent+"/src/main/java";
    System.out.println(parent);
    String child=packageName.replaceAll("\\.", "/");
    File f=new File(parent ,child);
    for(File c:f.listFiles()){
    if(c.getName().endsWith(".java")){
    list.add(packageName+"."+c.getName().substring(0,c.getName().lastIndexOf(".")));
    }
    }
    };
    public List<String > getList(){
    return list;
    }

{% endcodeblock %}

- 创建对应类的BeanFactory
- BeanFactory通过反射创建默认class默认实例对象
- 把创建完的对象放入ApplicationContext持有的对象Map
- 遍历beanFactory的Map，调用setProperties接口来完成依赖注入

{% codeblock  lang:java   %}

    private void init() throws ClassNotFoundException{
    compentScan.scan(backPackage);
    for(String clasName:compentScan.getList()){
    Class cls=Class.forName(clasName);
    Compent c=(Compent) cls.getAnnotation(Compent.class);
    if(c!=null){
    String key=c.value();			
    BeanFactory bf=new SimpleBeanFactory(cls);
    beanFactorys.put(cls, bf);
    map.put(key, bf.getBean());
    }
    }
    for(Entry<Class, BeanFactory> entry :beanFactorys.entrySet()){
    entry.getValue().setProperties(map);
    }
    
    }

{% endcodeblock %}

- 先创建对象，再做依赖注入 
- 容器里面是原生对象还是代理对象，取决于beanfatory返回的对象实例

