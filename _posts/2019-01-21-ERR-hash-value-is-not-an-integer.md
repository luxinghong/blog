---
layout: post
title: "ERR hash value is not an integer"
subtitle: ""
author: ""
header-style: text
tags:
  - redis

---



在用redis hash结构的increment功能时报了这个错。

一般hash结构都是采用JsonSerialize或JdkSerialize来对对象进行序列化，而有时我们不光想存对象，也会存一些简单的值。increment操作不会反序列化而是直接对存储的值进行操作，此时该值经过Json或Jdk序列化后不能直接进行加减操作，因为不能转换为对应的类型如integer、double、long。

最简单的办法就是重写JsonSerialize的serialize方法，加上一个判断，如果是String类型，就直接返回String.getBytes()，对象还是走原来的序列化方式。

![](/blog/img/20190120235747449.png)


这样既能用序列化方式存储对象，也能用字节存储简单值从而进行加减操作。