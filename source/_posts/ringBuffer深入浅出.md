---
title: ringBuffer深入浅出
date: 2023-06-20 15:55:40
tags:
---
#ringBuffer深入浅出

ringBuffer顾名思义就是环形缓冲区，常用来做高速缓冲队列，采用环形复用(缓存区写满之后，从首地址重新写入)，使用较小的实际物理内存实现了线性缓存。例如著名的Disruptor高性能的主要原因就是使用了ringBuffer。

##ringBuffer特性
- 存在读写2个序号
- 读/写序号一直累加
- 读/写序号取模buffer长度等于当前当前读写指针位置
- buffer里面的数据不需要删除，覆盖即可
- ringBuffer采用数组实现，对cpu高速缓存友好（cache line会加载相邻元素，数组元素天生相邻），访问速度比链表快。

##RingBuffer代码

{% codeblock  lang:java   %}


    package wiki.chenxun.study.buffer;
    import sun.misc.Unsafe;
    import java.lang.reflect.Field;
    import java.security.AccessController;
    import java.security.PrivilegedExceptionAction;
    import java.util.concurrent.TimeUnit;
    /**
     * @author chenxun
     * @date 2019-11-19
     * @since
    */
    public class RingBuffer<T> {
    private static Unsafe unsafe;
    private final T[] buffer;
    private volatile int readIndex =-1;
    private volatile int writerIndex =-1;
    private static long readIndexOffset;
    private static long writerIndexOffset;
    static {
    try {
    unsafe = AccessController.doPrivileged((PrivilegedExceptionAction<Unsafe>) () -> {
    Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
    theUnsafe.setAccessible(true);
    return (Unsafe) theUnsafe.get(null);
    });
    readIndexOffset = unsafe.objectFieldOffset
    (RingBuffer.class.getDeclaredField("readIndex"));
    writerIndexOffset = unsafe.objectFieldOffset
    (RingBuffer.class.getDeclaredField("writerIndex"));
    } catch (Exception e) {
    e.printStackTrace();
    }
    }
    @SuppressWarnings("all")
    public RingBuffer(int bufferSize) {
    buffer = (T[])new Object[bufferSize];
    }
    /**
      * 队列添加元素，如果队列满了，则等待到队列空余
      * @param t
      * @throws InterruptedException
        */
        public void push(T t) throws InterruptedException {
        while (true){
        //判断是否可写 总长度-（写位置-读位置）
        if(buffer.length-(writerIndex-readIndex)>0){
        int nextWriterIndex=writerIndex+1;
        boolean result=unsafe.compareAndSwapInt(this,writerIndexOffset,writerIndex,nextWriterIndex);
        if(result){
        //计算实际位置
        int bufferIndex=writerIndex%buffer.length;
        buffer[bufferIndex]=t;
        break;
        }
        }
        // 休眠1ms
        TimeUnit.MILLISECONDS.sleep(1L);
        }
        }
        /**
      * 队列取出元素，如果没有可读元素，则等待到有可读元素
      * @return
      * @throws InterruptedException
        */
        public T pop() throws InterruptedException {
        while (true){
        // 判断是否可读
        if(writerIndex-readIndex>0){
        int nextReadIndex=readIndex+1;
        boolean result=unsafe.compareAndSwapInt(this,readIndexOffset,readIndex,nextReadIndex);
        if(result){
        //计算实际位置
        int bufferIndex=readIndex%buffer.length;
        return buffer[bufferIndex];
        }
        }
        // 休眠1ms
        TimeUnit.MILLISECONDS.sleep(1L);
        }
        }
        }

{% endcodeblock %}

##测试代码
{% codeblock  lang:java   %}

    package wiki.chenxun.study.buffer;
    import java.io.IOException;
    import java.util.Random;
    import java.util.concurrent.TimeUnit;
    /**
    * @author chenxun
    * @date 2019-11-19
     * @since
    */
    public class RingBufferTest {
    public static void main(String[] args) {
    RingBuffer<String> ringBuffer=new RingBuffer<>(20);
    for(int a=1;a<4;a++){
    final int b=a;
    new Thread(()->{
    for(int i=0;i<10;i++){
    try {
    ringBuffer.push(String.valueOf(b*10+i));
    } catch (InterruptedException e) {
    e.printStackTrace();
    }
    try {
    Random r=new Random();
    TimeUnit.MILLISECONDS.sleep(r.nextInt(5));
    } catch (InterruptedException e) {
    e.printStackTrace();
    }
    }
    }).start();
    }
    for(int a=0;a<2;a++){
    new Thread(()->{
    while (true){
    String s= null;
    try {
    s = ringBuffer.pop();
    System.out.println(s);
    } catch (InterruptedException e) {
    e.printStackTrace();
    }
    }
    }).start();
    }
    try {
    System.in.read();
    } catch (IOException e) {
    e.printStackTrace();
    }
    }
    }

{% endcodeblock %}

##buffer下标计算优化
如果buffer的size是2的n次方，那么取模运算可以替换成位运算.代码参考如下
{% codeblock  lang:java   %}


    package wiki.chenxun.study.buffer;

    /**
    * @author chenxun
    * @date 2019-11-19
    * @since
    */
    public class NumberUtils {

    public static int moduloOperationOn2(int a, int b) {
    return a & b - 1;
    }

    public static void main(String[] args) {
    int a=100%8;
    int b=moduloOperationOn2(100,8);
    System.out.println(a==b);

    }
    }


{% endcodeblock %}

##缓存伪共享优化
cpu加载数据是基于缓存行（具体原理不展开了），java8支持通过@sun.misc.Contended来解决这个问题，
非java8版本可以参考Disruptor的做法 https://github.com/LMAX-Exchange/disruptor/blob/master/src/main/java/com/lmax/disruptor/RingBuffer.java

##读写序号java溢出问题
暂时没有想到好方案，目前大致思路是，设置一个阀值算法，满足阀值则重置大小，类似java集合扩容一样，但这个会导致读写锁住。

##RingBuffer应用场景

- 依赖一个优惠券的查询接口，假设该接口提供分页查询能力。单次最多返回100张优惠优惠券，单次rt是20ms
- 优惠券的扩展信息是另一个接口，该接口性能提供批量查询能力，单次做多支持30张优惠券，单词rt是30ms
- 假设某个用户有500张优惠券

如果是串行调用，则整体时间是 (500/100)20+(500/30)30.采用ringBuffer解耦两个接口依赖，则优化后的rt时间，
理论上可以缩减至30ms左右（排除cpu切换上下文损耗等）
