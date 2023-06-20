---
title: java线程状态
date: 2023-06-20 14:46:26
tags:
---
#java线程状态
java线程有以下几个状态

- new 初始状态
- runnable 运行状态（包括就绪和运行）
- blocked 阻塞状态
- waiting 等待状态,需要其他线程特定动作唤醒（通知或者中断）
- time-waiting 超时等待状态，可以在指定时间内返回。
- terminated 终止状态

##创建
java构造线程有两种办法,Thread类和Runnable接口,Thread的类本身实现了Runnable接口。
常见的作法,是通过thread类的public Thread(Runnable target)构造函数来创建线程。

{% codeblock  lang:java   %}

    new Thread(new Runnable() {
	@Override
	public void run() {
		// TODO Auto-generated method stub
		
	}
    });

{% endcodeblock %}

thread类的初始话代码
{% codeblock  lang:java   %}

    private void init(ThreadGroup g, Runnable target, String name,
    long stackSize, AccessControlContext acc) {
    if (name == null) {
    throw new NullPointerException("name cannot be null");
    }
    this.name = name;
    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    if (g == null) {
    if (security != null) {
    g = security.getThreadGroup();
    }

        if (g == null) {
            g = parent.getThreadGroup();
        }
    }
   
    g.checkAccess();
 
    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }
    g.addUnstarted();
    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext =
            acc != null ? acc : AccessController.getContext();
    this.target = target;
    setPriority(priority);
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    this.stackSize = stackSize;
   
    tid = nextThreadID();
    }


{% endcodeblock %}

从这段代码可以看出以下几点
当前线程为创建线程的父线程
child线程继承parent的Daemon、优先级、和加载资源的contextClassLoader和一个可以继承的ThreadLocal。
**Daemon线程（后台线程）的finally块代码不会执行**

##运行
Thread类的start方法起来运行线程。底层通过调用navate方法来运行线程。
**start（）方法来启动线程，真正实现了多线程运行，这时无需等待run方法体代码执行完毕而直接继续执行下面的代码**
**run（）方法当作普通方法的方式调用，程序还是要顺序执行，还是要等待run方法体执行完毕后才可继续执行下面的代码**

##阻塞
线程执行到monitorenter保护的代码快时，会去获取monitorenter对应的对象锁，如未获取，则该线程进入同步队列，状态为阻塞。
**任意对象都拥有自己的monitor，包括class对象。**

##等待/唤醒
线程等待的方法主要有下面几种
- object.wait() / object.wait(long)
- object.join() / object.join(long)
- lockSupport.park() / lockSupport.park(long)

当线程调用某个对象的wati方法，进入等待。当另一个线程调用该对象的notify()/notifyall()方法，之前等待的线程唤醒。
线程唤醒的方法注意下面几种
- object.notify()/notifyAll()
- lockSupport.unpark(Thread)


**调用对象wait或者notify方法的前题是获取当前对象的内置锁。**

通过wait/notify设计的经典作法
等待方：
- 获取对象锁
- 条件检查
- 为假则等待
- 为真则执行后面的逻辑
唤醒方：
- 获取对象锁
- 改变条件
- 通知所有等待在对象的线程

**条件变量用volatile修饰来保证可见性**

##终止
线程对应的run方法运行完毕，则线程终止。不推荐使用stop来终止线程。可以通过状态变量来终止线程。


