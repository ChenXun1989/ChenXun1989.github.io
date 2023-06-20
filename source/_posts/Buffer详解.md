---
title: Buffer详解
date: 2023-06-20 14:38:07
tags:
 - java源码
---
# Buffer详解

java中的nio由channel,buffer,Selector组成核心API。
Buffer是一个继承于object的抽象类,主要有下面几种。

- ByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer

基本覆盖了基本类型的支持，看下byteBuffer的实现

{% codeblock  lang:java   %}

    // 间接缓冲区
    ByteBuffer bf1=ByteBuffer.allocate(16);
    // 直接缓冲区
    ByteBuffer bf2=ByteBuffer.allocateDirect(16);

{% endcodeblock %}
ByteBuffer提供两种方式来获取缓存区内存，间接或者直接。
字节缓冲区要么是直接的，要么是非直接的。
如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在此缓冲区上执行本机 I/O 操作。
也就是说，在每次调用基础操作系统的一个本机 I/O 操作之前（或之后），
虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。
**直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，不会被GC回收。**
间接缓冲区与本地操作系统低层次的i/o交互需要一个中间缓冲区来实现，因此性能比直接缓存区要低。
直接缓冲区通过sun.misc.Unsafe类调用jni本地方法来分配内存。
因为绕过jvm,因此分配内存的开销较大，并且需要用户自己当成普通资源来主动释放。
堆外内存的分配

{% codeblock  lang:java   %}

    DirectByteBuffer(int cap) {                   // package-private
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);
    long base = 0;
    try {
    base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
    Bits.unreserveMemory(size, cap);
    throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
    // Round up to page boundary
    address = base + ps - (base & (ps - 1));
    } else {
    address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
    }

{% endcodeblock %}

可以看出通过unsafe.allocateMemory方法来分配堆外内存，同时创建一个对应的Cleaner
sun.misc.Cleaner.create方法两个参数。
- 需要监控的堆内存对象，就是堆外内存关联的对象。
- 程序释放资源的回调。

当JVM进行GC的时候，如果发现我们监控的对象，不存在强引用了，(Cleaner本身是幽灵引用)，
就会调用预先绑定的回调方法，我们一般在回调方法里面释放堆外内存

{% codeblock  lang:java   %}

    private static class Deallocator
    implements Runnable
    {
    private static Unsafe unsafe = Unsafe.getUnsafe();
    private long address;
    private long size;
    private int capacity;
    private Deallocator(long address, long size, int capacity) {
    assert (address != 0);
    this.address = address;
    this.size = size;
    this.capacity = capacity;
    }
    public void run() {
    if (address == 0) {
    // Paranoia
    return;
    }
    unsafe.freeMemory(address);
    address = 0;
    Bits.unreserveMemory(size, capacity);
    }
    }


{% endcodeblock %}

最终通过unsafe类提供freeMemory来释放之前分配的对外内存。
**堆外内存大小可以通过jvm启动参数限制： -XX:MaxDirectMemorySize**