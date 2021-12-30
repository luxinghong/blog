---
layout: post
title: "ThreadLocal"
subtitle: ""
author: ""
header-style: text
tags:
  - 多线程
---



# 作用

同一个 `ThreadLocal` 对象可以管理多个线程中的对象，且线程之间数据是隔离的，互不干扰。

有两个主要应用场景：

- 每个线程需要自己拥有一份独立的对象，比如 `SimpleDateFormat`，它是线程不安全的
- 在同一线程中传递全局变量，比如会话信息、用户权限等等，就不需要为每个方法都加一个入参了

------

<br/><br/>





# 示例

演示一下全局变量的用法。

```java
public class UserHolder{
    public static final ThreadLocal<User> USER_HOLDER = new ThreadLocal<>();
}

//以下方法在不同类中

public void method1(Long id){
    User user = findUserById(id);
    UserHolder.USER_HOLDER.set(user);
}

public void method2(){
    User currentUser = UserHolder.USER_HOLDER.get();
    //业务逻辑
}

public void method3(){
    User currentUser = UserHolder.USER_HOLDER.get();
    //业务逻辑
}
```

只要是在同一个线程中，`method2`、`method3`  取到的都是同一个用户。

不同线程取到的是不同用户。

------

<br/><br/>





# 原理

**本质：每个线程会维护自己的一个 map：`ThreadLocalMap`**：

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```



看下 ThreadLocal set() 方法的源码：

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

```java
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

`set` 时会先获取当前线程，然后取当前线程中的 `ThreadLocalMap`，把值 set 进去。

每个线程使用 `threadLocal.get()` 时是在自己的map中获取，就是这样做到线程隔离的。



可看到set时，key 是 `this`，即 threadLocal 对象，这么设计是因为 `threadLocalMap` 每个线程只有一个，而一个线程可以有多个 `threadLocal` 对象，如果需要多个全局变量的话。



再继续看 `threadLocalMap` 中的 `set` 方法：

```java
private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

`ThreadLocalMap` 类似于 HashMap，不同在于 `ThreadLocalMap` 只有数组(初始容量为16)，没有链表，通过 `key.threadLocalHashCode & (len-1)` 计算出数组下标存储位置。

那没有链表，如何解决冲突的呢？

其实很简单，就往后顺延就可以了。如果下标位置中没值，直接存储。如果有值，判断 key 是否相等，相等的话刷新，不等的话顺延一个位置重复之前的步骤。

`ThreadLocalMap` 有一个阈值：数组容量的$2/3$，当超过这个阈值的时候会进行扩容，扩容为原来容量的2倍。

由于 `ThreadLocalMap` 解决冲突的办法是顺延法，相对于 HashMap 的链表法在大数据量时效率较低。所以它不适合用来存储大量的值，它的设计初衷也只是为了解决全局变量传递的问题，并不是用来做一个容器。

------

<br/><br/><br/>









# 内存泄露问题

内存泄露指的是已分配出去的内存，回收不了，就不能再利用，这块内存就像凭空消失了一样。内存泄露积累下来，还会造成内存空间耗尽，导致 OOM。



先看一下 `ThreadLocalMap` 的 `Entry` 的数据结构：

```java
static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

可看到 `Entry` 是一个 `WeakReference`，在介绍 `WeakReference`  之前先说下垃圾回收。

 JVM 的垃圾回收算法有两种：引用计数法和可达性分析算法，从各自名字就能看出大概是什么意思。目前主流垃圾回收器采取的是可达性分析算法，该算法的本质是会从一系列 "GC Roots" 合集出发，探索所有能被该合集引用到的对象，并将其加入到该集合中，这个过程称为 “标记”，然后继续探索。最终，未被探索到的对象便是 “死亡” 对象，便会被垃圾回收器回收。

所以一个对象会不会被 GC 就看它能不能被探索到，换句话说有没有引用指向它，是不是可达的。**GC 的工作会由 JVM 自动来完成，但程序员也可以通过代码的方式（通过定义不同类型的对象引用 ）来影响一个对象的生命周期，这就是 `Reference` 的作用**。

Java 中的四种引用：

- 强引用
- 软引用
- 弱引用
- 虚引用

threadlocalmap 的 key 是一个弱引用，发生gc就会被回收，这样就出现了 key 为 null 的 entry，而如果使用的是线程池，线程一直存在，就会存在一个强引用：thread -> threadlocalmap -> entry -> value，导致 value 不能被回收。

使用完以后记得 remove，清理 key 为 null 的 entry