---
title: synchronized原理解析
date: 2023-06-20 14:30:26
tags:
---

#synchronized原理解析

##synchronized的三种方式
对于普通同步方法,锁的是当前实例对象
{% codeblock  lang:java   %}

    synchronized void test1(){
    //do something
    }
{% endcodeblock %}
对于静态方同步方法，锁的是当前类的Class对象

{% codeblock  lang:java   %}

    static synchronized void test2(){
    //do something
    }

{% endcodeblock %}

对于同步方法快，锁的是synchronized括号里的对象

{% codeblock  lang:java   %}

    void test3(){
	synchronized(this){
		//do something	
	}
    }

{% endcodeblock %}

##synchronized锁升级

jdk1.6之后，synchronized锁有四种状态，无锁，偏向锁，轻量级锁，重量级。

- 偏向锁： 存在java对象头里面。
- 轻量级锁：存在当前线程的栈帧里面
- 重量级锁： 存在当前对象的monitor对象里面

锁升级过程：
- 无锁状态到偏向锁，当前线程通过cas修改java对象头的锁标志位。
- 谝向锁到轻量级锁, 当前线程把java对象头的锁信息copy到栈帧里面
- 轻量级锁到重量级锁，把栈帧里面的锁信息copy到对象的monitor里面。
- 偏向锁和轻量级锁采用cas自旋修改锁状态，重量级锁通过线程信号来实现*
