---
layout: post
title:  "StringBuilder在高性能场景下的正确用法"
date:   2016-02-25 21:00:13
categories: java
permalink: /blogs/stringBuilder
---

   关于StringBuilder，一般同学只简单记住了，字符串拼接要用StringBuilder，不要用＋，也不要用StringBuffer.....?
还有些同学，还听过三句似是而非的经验：
1. Java编译优化后＋和StringBuilder的效果一样；
2. StringBuilder不是线程安全的，为了“安全”起见最好还是用StringBuffer；
3. 永远不要自己拼接日志信息的字符串，交给slf4j来。
 
* 1. 初始长度好重要，值得说四次。StringBuilder的内部有一个char[]， 不断的append()就是不断的往char[]里填东西的过程。
new StringBuilder() 时char[]的默认长度是16，然后，如果要append第17个字符，怎么办？
用System.arraycopy成倍复制扩容！！！！
这样一来有数组拷贝的成本，二来原来的char[]也白白浪费了要被GC掉。可以想见，一个129字符长度的字符串，经过了16，32，64, 128四次的复制和丢弃，合共申请了496字符的数组，在高性能场景下，这几乎不能忍。
所以，合理设置一个初始值多重要。
但如果我实在估算不好呢？多估一点点好了，只要字符串最后大于16，就算浪费一点点，也比成倍的扩容好。
 
* 2. Liferay的StringBundler类Liferay的StringBundler类提供了另一个长度设置的思路，它在append()的时候，不急着往char[]里塞东西，而是先拿一个String[]把它们都存起来，到了最后才把所有String的length加起来，构造一个合理长度的StringBuilder。
 
* 3. 但，还是浪费了一倍的char[]浪费发生在最后一步，StringBuilder.toString()
  // Create a copy, don't share the array
return new String(value, 0, count);
String的构造函数会用 System.arraycopy()复制一把传入的char[]来保证安全性不可变性，如果故事就这样结束，StringBuilder里的char[]还是被白白牺牲了。
为了不浪费这些char[]，一种方法是用Unsafe之类的各种黑科技，绕过构造函数直接给String的char[]属性赋值，但很少人这样做。
另一个靠谱一些的办法就是重用StringBuilder。而重用，还解决了前面的长度设置问题，因为即使一开始估算不准，多扩容几次之后也够了。
 
* 4. 重用StringBuilder这个做法来源于JDK里的BigDecimal类（没事看看JDK代码多重要），SpringSide里将代码提取成StringBuilderHolder，里面只有一个函数
 ```
  public StringBuilder getStringBuilder() {
sb.setLength(0);
return sb;
}
```
StringBuilder.setLength()函数只重置它的count指针，而char[]则会继续重用，而toString()时会把当前的count指针也作为参数传给String的构造函数，所以不用担心把超过新内容大小的旧内容也传进去了。可见，StringBuilder是完全可以被重用的。
为了避免并发冲突，这个Holder一般设为ThreadLocal，标准写法见BigDecimal或StringBuilderHolder的注释。
 
* 5. ＋ 与 StringBuilder  String s ＝ “hello ” + user.getName();
这一句经过javac编译后的效果，的确等价于使用StringBuilder，但没有设定长度。
```
  String s ＝ new StringBuilder().append(“hello”).append(user.getName());
  ```
但是，如果像下面这样：
```
  String s ＝ “hello ”;
// 隔了其他一些语句
s = s ＋ user.getName();
```
每一条语句，都会生成一个新的StringBuilder，这里就有了两个StringBuilder，性能就完全不一样了。如果是在循环体里s+=i; 就更加多得没谱。
据R大说，努力的JVM工程师们在运行优化阶段， 根据+XX:+OptimizeStringConcat(JDK7u40后默认打开)，把相邻的(中间没隔着控制语句) StringBuilder合成一个，也会努力的猜长度。
所以，保险起见还是继续自己用StringBuilder并设定长度好了。
 
* 6. StringBuffer 与 StringBuilderStringBuffer与StringBuilder都是继承于AbstractStringBuilder，唯一的区别就是StringBuffer的函数上都有synchronized关键字。
那些说StringBuffer “安全”的同学，其实你几时看过几个线程轮流append一个StringBuffer的情况？？？
 
* 7. 永远把日志的字符串拼接交给slf4j??  logger.info("Hello {}", user.getName());
对于不知道要不要输出的日志，交给slf4j在真的需要输出时才去拼接的确能省节约成本。
但对于一定要输出的日志，直接自己用StringBuilder拼接更快。因为看看slf4j的实现，实际上就是不断的indexof("{}"), 不断的subString()，再不断的用StringBuilder拼起来而已，没有银弹。
PS. slf4j中的StringBuilder在原始Message之外预留了50个字符，如果可变参数加起来长过50字符还是得复制扩容......而且StringBuilder也没有重用。
 
* 8. 小结
   StringBuilder默认的写法，会为129长度的字符串拼接，合共申请625字符的数组。所以高性能的场景下，永远要考虑用一个ThreadLocal 可重用的StringBuilder。由于线程私有之后，单个线程执行就是串行的，顺序执行的，就可以在用到StringBuilder时
取出线程私有的StringBuilder，也不会产生线程安全问题，同时减少了String对象的创建次数，故而提高性能。
