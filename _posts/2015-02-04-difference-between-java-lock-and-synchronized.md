---
layout: post
title: synchronized与lock的区别
category: java
tags: [lock,synchronized]
---

###几个概念

* 共享变量(shared variable)：多个线程都能访问的变量。

* 变量可见性(variable visibility)：一个线程更新了共享变量，对其他线程立刻可见。

* 互斥(mutual exclusion )：几个线程中的任何一个不能与其他一个或多个同时操作一个变量。

* 临界区(critical section)：访问共享资源的一段代码块。

###synchronized解决的问题

* 保证共享变量的可见性：变量缓存与编译器指令优化 会导致 变量修改的不可见性。

>The Java virtual machine (JVM) supports this mechanism via monitors and the associated monitorenter and monitorexit instructions.

* 保证共享变量的互斥性：同一时刻只能有一个线程对共享变量的读写。monitor lock实现，monitor提供了一种互斥访问的机制，保证只有一个进程进入临界区。

###synchronized与lock的区别

自jdk1.5后java.util.concurrent包提供了更广泛通用的锁实现lock，名字上的区别是，一个是隐式锁(intrinsic locking)，一个是显示锁(explicit lock)，从这点上来说，用户对显示锁有更多地控制。具体区别有以下几点：

1）lock提供了如下的方法：

* void lock()，获取一个锁，如果锁当前被其他线程获得，当前的线程将被休眠。
* boolean tryLock()，尝试获取一个锁，如果当前锁被其他线程持有，则返回false，不会使当前线程休眠。
* boolean tryLock(long timeout,TimeUnit unit)，如果获取了锁定立即返回true，如果别的线程正持有锁，会等待参数给定的时间，在等待的过程中，如果获取了锁定，就返回true，如果等待超时，返回false。
* void lockInterruptibly()，如果获取了锁定立即返回，如果没有获取锁定，当前线程处于休眠状态，直到或者锁定，或者当前线程被别的线程中断。

可见lock比synchronized提供了更细的粒度、更灵活的控制。

2） synchronized是在JVM层面上实现的，如果代码执行时出现异常，JVM会自动释放monitor锁。而lock代码是用户写的，需要用户来保证最终释放掉锁。通常使用lock的代码需要这样：

    Lock l = ...; // 获取锁
    l.lock();
    try 
    {
      // access the resource protected by this lock
    } catch (Exception ex) 
    {
      // restore invariants
    }
    finally 
    {
       l.unlock();
    }

3）lock提供了一个重要的方法newConditon()，ConditionObject有await()、signal()、signalAll()，类似于Ojbect类的wait()、notify()、notifyAll()。这些方法都是用来实现线程间通信。lock将synchronized的互斥性和线程间通信分离开来，一个lock可以有多个condition。另外lock的signal可以实现公平的通知，而notify是随机从锁等待队列中唤醒某个进程。

4）性能上来说，在多线程竞争比较激烈地情况下，lock比synchronize高效得多。如何说明一个锁更高效？

>A better lock implementation makes fewer system calls, forces fewer context switches, and initiates less memory-synchronization traffic on the shared memory bus, operations that are time-consuming and divert computing resources from the program.

###参考

1、[java-101-the-next-generation-java-concurrency-without-the-pain-part-2](http://www.javaworld.com/article/2078848/java-concurrency/java-101-the-next-generation-java-concurrency-without-the-pain-part-2.html)

2、[Choosing Between Synchronized and ReentrantLock](http://my.safaribooksonline.com/book/programming/java/0321349601/explicit-locks/ch13lev1sec4)







