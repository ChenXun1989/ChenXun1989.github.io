---
title: 详解CopyOnWriter
date: 2023-06-20 15:12:34
tags:
- java
---
# 详解CopyOnWriter

**示例代码链接 https://github.com/ChenXun1989/study-example**
copyOnWriter顾名思义就是写的时候复制，是一种常见的计算机程序设计领域的优化策略。
核心思想是当多个调用者访问同一个资源的时候(如内存或磁盘上的数据存储),他们会共同获取相同的指针指向同一个的资源。
当每个调用者修改资源的时候,系统才会真正复制一份专用副本给该调用者，而其他调用者所见到的最初的资源仍然保持不变.
专用副本修改完毕之后，替代真正的资源。最终所有访问者看到一致的视图。

## CopyOnWriterArrayList的实现细节

CopyOnWriterArrayList.get()方法

{% codeblock  lang:java   %}

    /**
     * {@inheritDoc}
     * 
     * @throws IndexOutOfBoundsException {@inheritDoc}
    */
    public E get(int index) {
    return get(getArray(), index);
    }

     /**
     *Gets the array.  Non-private so as to also be accessible
     * from CopyOnWriteArraySet class.
    */
    final Object[] getArray() {
    return array;
    }

    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;

{% endcodeblock %}

从源码上可以看出CopyOnWriterArrayList底层使用一个由volatile修饰的array数组来存值，
而读一个volatile变量（相当于获取锁），JMM会把线程对应的本地内存置为无效，
从而使得被监视器保护的临界区代码必须要从主内存中去读取共享变量。这样来保证读的变量是最新的。

CopyOnWriterArrayList.add()方法

{% codeblock  lang:java   %}

    /**
     * Appends the specified element to the end of this list.
     * 
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
    */
    public boolean add(E e) {
    //获取写锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
    // 获取真正的资源
    Object[] elements = getArray();
    int len = elements.length;
    // 复制一份私有副本
    Object[] newElements = Arrays.copyOf(elements, len + 1);
    // 修改私有副本
    newElements[len] = e;
    // 私有副本修改完毕之后，替代真正的资源
    setArray(newElements);
    return true;
    } finally {
    lock.unlock();
    }
    }

    /**
     * Sets the array.
    */
    final void setArray(Object[] a) {
    array = a;
    }

{% endcodeblock %}

从源码上可以看出CopyOnWriterArrayList里面add方法体现了CopyOnWriter的核心思想，具体步骤如下：
1. 获取写锁
2. 从原资源复制一份私有副本
3. 修改私有副本
4. 私有副本替换原资源
5. 释放写锁

## copyOnWriter的其他应用

CopyOnWriter是一种常见的计算机程序设计领域的优化策略，例如redis里面的持久化机制
redis处理数据是通过单线程的，但是很多场景我们需要持久化数据，reids则依赖系统的fork()函数的copyOnWriter实现。
redis官方文档：
> Whenever Redis needs to dump the dataset to disk, this is what happens:
> Redis forks. We now have a child and a parent process.
> The child starts to write the dataset to a temporary RDB file.
> When the child is done writing the new RDB file, it replaces the old one.
> This method allows Redis to benefit from copy-on-write semantics.

liunx系统的fork函数实现方式使用copyOnWriter技术创建子进程。
当父进程创建子进程时，内核只为子进程创建虚拟空间，父子两个进程使用的是相同的物理空间。只有父子进程发生更改时才会为子进程分配独立的物理空间。

## copyOnWriter的优缺点
1. copyOnWriter适用读多写少的场景
2. 数据复制过程带来了更多得io消耗




