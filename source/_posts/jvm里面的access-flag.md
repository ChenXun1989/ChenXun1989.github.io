---
title: jvm里面的access_flag
date: 2023-06-20 15:30:09
tags:
---

#jvm里面的access_flag

##access_flag
最近在看jvm相关知识，发现class字节码文件里面大量使用access_flag做标志位，其中做法非常nice。
例如class字段描述，java里面字段修饰符有以下几个

- public
- private
- protected
- static
- final
- volatile
- transient

{% codeblock lang:java   %}

    private static final  String a="abc";
{% endcodeblock %}

这段代码在class文件里面字段a的访问修饰符怎么描述的呢，答案是 1A（实际是2进制值，方便阅读转成16进制）

#fields的access_flag规范
| 标志名称 | 标志值（16进制） | 含义|
| ---    | ---    | --- |
|ACC_PUBLIC	|0x0001	|字段是否为public|
|ACC_PRIVATE	|0x0002	|字段是否为private|
|ACC_PROTECTED	|0x0004	|字段是否为protected|
|ACC_STATIC	|0x0008	|字段是否为static|
|ACC_FINAL	|0x0010 |字段是否为final|
|ACC_VOLATILE	|0x0040	|字段是否为volatile|
|ACC_TRANSIENT	|0x0080	|字段是否为transient|
|ACC_SYNTHETIC	|0x1000	|字段是否为编译器自动产生|
|ACC_ENUM	|0x4000	|字段是否为enum|

看完规范发现，1A这个值完全不在规范里面，那jvm怎么知道1A就是private static final 呢？

##access_flag的秘密
我们把相关标志16进制转成二进制看下

|16进制	|二进制 |
| --- | --- |
|0x0001	|000001|
|0x0002	|000010|
|0x0004	|000100|
|0x0008	|001000|
|0x0010	|010000|

童鞋们发现规律了吗？为什么二进制值是000001，000010 ，每一个标志独占一位。这样标志值就满足以下规律
a(标志值)|b（标志值）=c （class文件上的值）
jvm怎么处理的呢？
c & a != 0,则表示该字段有a修饰符

##业务应用
access_flag的秘密其实非常简单，了解之后怎么应用到业务场景呢
- 有限的值
- 不同值组合关系是要么全是并且，要么全是或者
  例如 有一个优惠券，有N种使用条件，切每个条件必须同时满足才能使用
  此类场景就非常适合。



