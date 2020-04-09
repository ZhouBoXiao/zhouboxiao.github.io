---
6layout:     post
title:      "Java的锁机制"
subtitle:   "偏向锁、CAS、Synchronized、Lock等"
date:       2020-03-16
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - Java
---

~还待进一步完善~

​	

32位

| 锁状态   | 25bit                                          | 4bit         | 1bit是否是偏向锁 |
| -------- | ---------------------------------------------- | ------------ | ---------------- |
| GC标记   | 空                                             |              |                  |
| 重量级锁 | 指向重量级锁Monitor指针                        |              |                  |
| 轻量级锁 | 指向线程栈中锁记录的指针pointer to lock Record |              |                  |
| 偏向锁   | 线程ID、 Epoch                                 | 对象分代年龄 | 1                |
| 无锁     | 对象的hashCode                                 | 对象分代年龄 | 0                |



# Lock

## ReetrantLock

- 等待可中断
  - lock.lockInterruptibly()
- 可实现公平锁
  - ReentrantLock(boolean fair)
- 可实现选择性通知
  - 线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知。
- AQS
  - AQS是Abstract Queued Synchronizer，它是用来构建锁或其他同步组件的基础框架，内部通过一个int类型的成员变量state来控制同步状态，当state=0时，则说明没有任何线程占有共享资源的锁，当state=1时，则说明有线程目前正在使用共享变量，其他线程必须加入同步队列进行等待。
  - AQS内部通过**内部类Node构成FIFO的同步队列**来完成线程获取锁的排队工作，同时利用内部类**ConditionObject构建等待队列**，当Condition调用wait()方法后，线程将会加入等待队列中，而当Condition调用signal()方法后，线程将从等待队列转移动同步队列中进行锁竞争。
  - 涉及到两种队列，一种的同步队列，当线程请求锁而等待的后将加入同步队列等待，而另一种则是等待队列(可有多个)，通过Condition调用await()方法释放锁后，将加入等待队列。

## Synchronized

- 重要的三种使用方式
  - 修饰实例方法: 作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁
  - 修饰静态方法: 也就是给当前类加锁，会作用于类的所有对象实例，因为静态成员不属于任何一个实例对象，是类成员
  - 修饰代码块: 指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。
- synchronized 同步语句块的情况
  - 操作系统提供monitor机制，c++实现，mutex 机制- > pthread_mutex_lock (内核态)。
  - Object的wait/notify/notifyAll方法都依赖于monitor对象，monitor对象存在于每个Java对象的对象头中。
  - 翻译成指令是`monitorenter `、`monitorexit`。
  - Java对象存储在内存中，包含对象头、实例数据和对齐填充三部分，其中对象头包含锁标志位。
- synchronized 修饰方法的的情况
  - `ACC_SYNCHRONIZED `标识，该标识指明了该方法是一个同步方法，JVM 通过该 `ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

### 区别

1. 原始构成

   1. Synchronized是关键字属于JVM层面，Lock是API层面的锁

2. 使用方法

   1. Synchronized不需要用户手动释放锁，当Synchronized代码执行完后系统会自动让线程释放对锁的占用。ReentrantLock需要手动释放

3. 等待是否可中断

   1. Synchronized不可中断。

   2. ReentrantLock可中断，

      ```java
      //1、设置超时方法
      tryLock(long timeout, TimeUnit unit)
      //2、lockInterruptibly()放代码块中，调用interrupt()方法可中断
      ```

4. 加锁是否公平

   1. Synchronized非公平锁，ReentrantLock两者都可以，默认非公平锁，构造方法可以传入boolean值，true为公平锁，false为非公平锁。

5. 锁绑定多个条件Condition

   1. Synchronized没有，ReetrantLock用来实现分组唤醒需要唤醒的线程们，可以**精确唤醒**，而不是像Synchronized要么随机唤醒一个线程，要么唤醒全部线程。

## 偏向锁

- 偏向锁的目标是，减少无竞争且只有一个线程使用锁的情况下，使用轻量级锁产生的性能消耗。轻量级锁每次申请、释放锁都至少需要一次CAS，但偏向锁只有初始化时需要一次CAS。

- “偏向”的意思是，*偏向锁假定将来只有第一个申请锁的线程会使用锁*（不会有任何线程再来申请锁），因此，只需要在Mark Word中CAS记录owner（本质上也是更新，但初始值为空），如果记录成功，则偏向锁获取成功，记录锁状态为偏向锁，以后当前线程等于owner就可以零成本的直接获得锁；否则，说明有其他线程竞争，膨胀为轻量级锁。


## CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成操作。

```java
//一个或者多个线程，等待其他多个线程完成某件事情之后才能执行。
public class CountDownLatchTest { 
    staticCountDownLatch c = new CountDownLatch(2); 
    public static void main(String[] args) throws InterruptedException { 
        new Thread(new Runnable() {
            @Override 
            public void run() { 
                System.out.println(1); 
                c.countDown(); 
                System.out.println(2); 
                c.countDown();            
            }       
        }).start(); 
        c.await(); 
        System.out.println("3");    
    } 
}
```

## 同步屏障CyclicBarrier 

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一
组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会
开门，所有被屏障拦截的线程才会继续运行。

```
//多个线程互相等待，直到到达同一个同步点，再继续一起执行。

```

