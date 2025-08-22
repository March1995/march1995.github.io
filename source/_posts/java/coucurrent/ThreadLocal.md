---
title: ThreadLocal
date: 2019-03-02 
desc: java线程,多线程解析
keywords: thread
categories: [并发编程]
---

# 类图：

## ThreadLocal

![ThreadLocal](https://march1995.github.io/uploads/java/concurrent/ThreadLocal.png)

## Thread
![Thread](https://march1995.github.io/uploads/java/concurrent/Thread.png)

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
### 流程  先讲一下作用，再说自己看过源码，讲下源码

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

### 为什么 JDK 不把 value 也设计成弱引用？
如果 value 是弱引用：

只要 ThreadLocal 被回收，value 也会被回收。
但这样会导致 ThreadLocal 失去存储能力，因为 value 可能随时被 GC 回收，无法保证数据存活。

```java
ThreadLocal<byte[]> tl = new ThreadLocal<>();
tl.set(new byte[1024 * 1024]); // 1MB 数据

// 假设 value 是弱引用，且发生 GC：
System.gc();
byte[] data = tl.get(); // 可能返回 null，即使 tl 仍然存活！
```
JDK 的权衡：

key 弱引用：防止 ThreadLocal 对象本身泄漏。
value 强引用：确保数据可用，但要求开发者 必须手动 remove()

### JDK 的现有设计（key 弱引用 + value 强引用）

优点：

| 设计 | 回收时机 | 是否可靠 | 适用场景 |
|------|----------|---|-- |
|key弱引用 |  ThreadLocal 无强引用时回收 | ❌ | key 可能被回收	防止 ThreadLocal 对象本身泄漏 |
|value强引用 | 必须手动 remove() 或线程结束 | ✅ 数据可靠 | 保证业务逻辑正确性 |

平衡点：
key 弱引用 → 防止开发者忘记释放 ThreadLocal 对象（避免 ThreadLocal 本身泄漏）。
value 强引用 → 确保数据存活，由开发者控制清理（通过 remove()）。

缺点:
内存泄漏风险：
如果开发者忘记 remove()，且 ThreadLocal 被回收（key=null），value 会一直占用内存（直到线程结束，或 ThreadLocalMap 扩容时清理）。


### 为什么 key 弱引用 能 避免 ThreadLocal 本身泄漏

(1) 什么是 ThreadLocal 对象泄漏？

假设 ThreadLocal 是强引用：

```java
ThreadLocal<Object> threadLocal = new ThreadLocal<>();
threadLocal.set(new Object());
threadLocal = null; // 失去强引用
```
由于 ThreadLocalMap 的 Entry 对 ThreadLocal 是强引用，即使开发者已经不再使用 threadLocal 变量，ThreadLocal 对象仍然被 ThreadLocalMap 引用，无法被 GC 回收。

结果：ThreadLocal 对象本身泄漏（即使它已经不再需要）。

(2) 弱引用如何解决这个问题？

Entry 对 ThreadLocal（key）是弱引用：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value; // value 仍然是强引用
}
```
当 ThreadLocal 失去所有强引用（如 threadLocal = null），key 会被 GC 回收（因为它是弱引用），此时 Entry 的 key 变为 null。

效果：

ThreadLocal 对象可以被回收，避免自身泄漏。
但 value 仍然是强引用，需要手动 remove() 或依赖 ThreadLocalMap 的惰性清理。

## ThreadLocal内存溢出例子

```java
public class ThreadLocalMemoryLeakDemo {
    // 模拟一个大对象
    static class BigObject {
        private byte[] data;

        public BigObject() {
            // 每个对象占用约1MB内存
            this.data = new byte[1024 * 1024];
        }
    }

    // 使用静态ThreadLocal变量
    private static final ThreadLocal<BigObject> threadLocal = new ThreadLocal<>();

    /*
            -Xmx12M -Xms12M
     */
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 100; i++) {
            executor.execute(() -> {
                // 设置ThreadLocal值但不清理
                threadLocal.set(new BigObject());
                System.out.println(Thread.currentThread().getName() + " set value");

                // 模拟工作
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                // 这里故意不调用threadLocal.remove()
            });
            Thread.sleep(50);
        }

        executor.shutdown();
    }
}
```

### 问题分析：
1.线程池复用线程：我们创建了一个固定5个线程的线程池，这些线程会被反复使用

2.ThreadLocal未清理：每个任务执行时都会设置一个新的BigObject(约1MB)到ThreadLocal中，但从未调用remove()

3.内存泄漏过程：

- 第一次任务执行时，线程的ThreadLocalMap中存入一个Entry

- 任务完成后，线程返回线程池，但Entry仍然存在

- 下次该线程执行新任务时，又存入新的BigObject

- 由于线程被复用，旧的Entry不会被自动清理

4.最终结果：

- 每个线程的ThreadLocalMap中会积累多个BigObject

- 随着任务不断执行，内存占用持续增长

- 最终导致OutOfMemoryError

```java
pool-1-thread-1 set value
pool-1-thread-2 set value
pool-1-thread-3 set value
pool-1-thread-4 set value
pool-1-thread-5 set value
pool-1-thread-1 set value
pool-1-thread-2 set value
Exception in thread "pool-1-thread-3" java.lang.OutOfMemoryError: Java heap space
	at com.wyb.thread.base.threadLocal.ThreadLocalMemoryLeakDemo$BigObject.<init>(ThreadLocalMemoryLeakDemo.java:13)
	at com.wyb.thread.base.threadLocal.ThreadLocalMemoryLeakDemo.lambda$main$0(ThreadLocalMemoryLeakDemo.java:26)
	at com.wyb.thread.base.threadLocal.ThreadLocalMemoryLeakDemo$$Lambda$14/0x0000000800064440.run(Unknown Source)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
pool-1-thread-4 set value
pool-1-thread-5 set value
pool-1-thread-1 set value
pool-1-thread-6 set value
```