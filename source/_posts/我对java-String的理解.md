---
title: String
date: 2016-04-19 17:50:05
tags: java
---
##  我对java String的理解 
在java开发中，String几乎最常用的类型了。在系统中，字符串计算是十分耗费资源的。为此，sun在设计String是，采用了很多奇妙的设计。
## String的不可变性
在java设计中，String类型是不可变的。如
{%codeblock%}
   String s1 = "abc";
   String s2 = s1.toUpperCase();
{%endcodeblock%}
   实际上，s1依旧指向 abc ，而s2指 向新生成的ABC所在的新的地址。换句话说，String具有只读性。

## String的 + 
大家都知道，在java中，是不允许对像C++一样操作符重载的。但是，对于String来说有点例外，它重载了，“+”、“+=”两个操作符号

在日常程序编写中，我是经常会编写字符串拼接的程序。如
 String s  = "I"+"love"+"CS";
 按照String不变性在推测，是不是在生成新的s时，
 第一步：新生成"Ilove"对象
 第二部：生成"IloveCS"对象
 如果字符串拼接项很多，那么。那么中间就会生成很多对象。Gc也会不断的回收新生成的对象。在一个大型的程序中，如此低效率的行为，明显是不会被允许的。

 事实上，在java编译中，实际上采用了new StringBuilder的方式，优化了这个问题。
 上述过程，最终实现优化后，差不多如下
 {%codeblock%}
 StringBuilder sb  = new StringBuilder();
 sb.append("I");
 sb.append("love");
 sb.append("CS");
 String s = sb.toString();
{%endcodeblock%}
 注意：在字符串拼接十分复杂的情况下，需要自己生成StringBuilder。单纯依靠编译器优化。可能依旧存在效率问题。

 ## String 存储
 String是一个非常有意思的类。在内存中存储的方式不同
 当String s1= "abc" 时，String是存在静态区。且在静态区内，同一个字符串，在静态区，只能存有一份。

 {%codeblock%}
 String s2 = "a";
 String s3 = "a";
 s2 == s3 //  true
{%endcodeblock%}
 注意： == 比较的是内存地址是否相等。如果是字符串内容是否相等，则用equal（）

 {%codeblock%}
 String s4 = new String("abc");是生成在堆内存中。
 String s5 = new String("abc");
 s4 == s5 //false
{%endcodeblock%}
String s6 = "abc" + new String("cde"); 也是生成在堆内存中，因为new 后面只有在运行时才会被知道具体内容。



