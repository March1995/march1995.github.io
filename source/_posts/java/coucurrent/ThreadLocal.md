---
title: ThreadLocal
date: 2019-03-02 
desc: java线程,多线程解析
keywords: thread
categories: [JavaSE]
---

# 类图：

## ThreadLocal

![ThreadLocal](/source/uploads/java/concurrent/ThreadLocal.png)

## Thread
![Thread](/source/uploads/java/concurrent/Thread.png)

# 源码：
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

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
```
### Thread

```java
ThreadLocal.ThreadLocalMap threadLocals = null;

ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
# 流程  先讲一下作用，再说自己看过源码，讲下源码

ThreadLocal就是让线程有一份自己的数据副本，因此不需要考虑线程安全问题。
在我做过的项目中一般用来存储用户信息。因为后台中每一个请求都是一个线程，我们在后续需要这个数据的时候就可以很方便的获取这个信息。
ThreadLocal中有一个静态内部类ThreadLocalMap，这个Map里面的key是ThreadLocal这个对象，value就是我们要存储的值。
而ThreadLocalMap里面其实是维护了一个Entry数组，而Entry类是ThreadLocalMap的静态内部类，其中key存储的是弱引用的ThreadLocal，value存储的就是要存储的数据。
而在每条线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，
这个threadLocals就是每条线程用来存储数据副本的，key值为ThreadLocal对象，value为变量副本。
所以ThreadLocal和里面的静态内部类ThreadLocalMap其实是为每一个线程来服务的。在ThreadLocal源码中也可以看到，每次执行get和set方法的时候
，都要先获取当前访问的线程，然后再从当前线程获取该线程的ThreadLocalMap属性然后再来操作。

ThreadLocal可能出现的问题？
ThreadLocal会发生内存泄漏的问题。
ThreadLocalMap里面维护了一个Entry数组，而每个Entry类的key是弱引用的ThreadLocal，value是数据。
如果一个ThreadLocal对象没有外部的强引用来指向它，那么堆内存不足的时候会被GC掉这些弱引用的Key，那么就会出现key为null但是value不为null的情况，但是此时的value会一直存在一个强引用链，也就是Thread当前线程->ThreadLocalMap->Entry->Value，导致GC无法回收造成内存的泄漏，这个value就是泄漏的对象。
但是ThreadLocalMap在执行get、set、remove的时候会自动清理key为null的Entry，
但是更优雅的做法还是在使用完ThreadLocal之后调用ThreadLocal中的remove方法清空ThreadLocal变量副本解决该问题，
在我做的项目中，会在过滤器的最后执行ThreadLocal中的remove方法进行手动释放。
