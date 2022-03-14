---
layout: post
title: "ConcurrentHashMap"
subtitle: ""
author: ""
header-style: text
tags:
  - 多线程
---



# 为什么 HashMap 线程不安全？

HashMap 从设计之初就不是用在多线程环境下，内部所有操作都没有同步。

随便举两个例子：

- **哈希碰撞时值可能会被覆盖**

  以 `put` 为例，部分源码如下：

  ```java
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                 boolean evict) {
      Node<K,V>[] tab; Node<K,V> p; int n, i;
      if ((tab = table) == null || (n = tab.length) == 0)
          n = (tab = resize()).length;
      // 找到空的槽位，插入。
      if ((p = tab[i = (n - 1) & hash]) == null)
          tab[i] = newNode(hash, key, value, null);
  ```

  重点看注释后的两行。假设两个线程同时put，且它们的key产生的哈希值是一样的，即会发生哈希碰撞。

  由于没做任何同步，它们都可以通过 if 判断，都认为没有出现哈希碰撞，那么很明显，就会出现值被覆盖的情况。

  这是针对空的槽位的情况，对于已经有节点的槽位，HashMap 会采用尾插法将新节点赋给 p.next，也是一样的，会出现被覆盖的情况。

- **代码中随处可见的 ++ 自增操作**

  众所周知，`++` 自增操作非原子操作，线程不安全，依赖于此的逻辑自然也是线程不安全的。





# ConcurrentHashMap

还是以空槽位插入，作为对比，贴出部分代码：

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 找到空的槽位，插入。
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   
            }
```

使用了CAS来保证操作的原子性，如果 `casTabAt` 为 false 则表示有并发操作，重试（for循环）。

casTabAt 最终使用的 Unsafe.compareAndSwapObject 方法，是一个native方法，其最底层是依赖于CPU硬件提供的能力。

除此之外，还使用了 synchronized 来同步。（jdk1.6以后，JVM 对 synchronized 做了大量优化，synchronized 不再是重量级锁的代名词了）



**简而言之，ConcurrentHashMap 就是两个东西：CAS 和 synchronized。**