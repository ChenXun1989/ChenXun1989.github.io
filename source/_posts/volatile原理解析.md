---
title: volatile原理解析
date: 2023-06-20 14:18:09
tags:
---
#volatile原理解析

##volatile语义
jsr133规范对volatile语义进行了增强，明确保证了可见性和有序性。

**volatile不保证原子性**

java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致的更新，线程应该确保通过排他锁单独获得这个变量。
Java语言提供了volatile，在某些情况下比锁更加方便。
如果一个字段被声明成volatile，java线程内存模型确保所有线程看到这个变量的值是一致的。

**Volatile变量修饰符如果使用恰当的话，它比synchronized的使用和执行成本会更低，因为它不会引起线程上下文的切换和调度。**

##volatile的有序性

jmm为了保证volatile的语义中的有序性，通过内存屏障来禁止指令重排序
JMM基于保守策略的JMM内存屏障插入策略
- 在每个volatile写操作的前面插入一个StoreStore屏障
- 在每个volatile写操作的后面插入一个SotreLoad屏障
- 在每个volatile读操作的后面插入一个LoadLoad屏障
- 在每个volatile读操作的后面插入一个LoadStore屏障


x86处理器仅仅会对写-读操作做重排序,因此在x86中，JMM仅需在volatile后面插入一个StoreLoad屏障即可正确实现volatile写-读的内存语义 

##volatile的可见性
对于volatile修饰的变量，jvm虚拟机保证从主内存加载到线程工作内存的值是最新的。
从内存语义的角度来看
- volatile的写-读与锁的释放-获取有相同的内存效果
- volatile写和锁的释放有相同的内存语义
- volatile读与锁的获取有相同的内存语义


当线程释放锁的时候，JMM会把线程对应的本地内存中的共享变量刷新到主内存中
当线程获取锁时，JMM会把线程对应的本地内存置为无效，从而使得被监视器保护的临界区代码必须要从主内存中去读取共享变量。

##volatile的使用前提
- 对变量的写入操作不依赖变量的当前值，或者你能确保只有单个线程更新变量的值。
- 该变量没有包含在具有其他变量的不变式中。


