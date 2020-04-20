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

## 悲观锁与乐观锁

- 悲观锁
  - 总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁（**共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程**）。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。Java中`synchronized`和`ReentrantLock`等独占锁就是悲观锁思想的实现。

- 乐观锁
  - **写比较少的情况下（多读场景）**
  - 总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。**乐观锁适用于多读的应用类型，这样可以提高吞吐量**，像数据库提供的类似于**write_condition机制**，其实都是提供的乐观锁。在Java中`java.util.concurrent.atomic`包下面的原子变量类就是使用了乐观锁的一种实现方式**CAS**实现的。
  - 缺点
    - ABA 问题
    - 循环时间长开销大
    - 只能保证一个共享变量的原子操作

**CAS适用于写比较少的情况下（多读场景，冲突一般较少），synchronized适用于写比较多的情况下（多写场景，冲突一般较多）**

## 32位

| 锁状态   | 25bit                                          | 4bit         | 1bit是否是偏向锁 |
| -------- | ---------------------------------------------- | ------------ | ---------------- |
| GC标记   | 空                                             |              |                  |
| 重量级锁 | 指向重量级锁Monitor指针                        |              |                  |
| 轻量级锁 | 指向线程栈中锁记录的指针pointer to lock Record |              |                  |
| 偏向锁   | 线程ID、 Epoch                                 | 对象分代年龄 | 1                |
| 无锁     | 对象的hashCode                                 | 对象分代年龄 | 0                |

## volatile

例如你让一个volatile的integer自增（i++），其实要分成3步：1）读取volatile变量值到local； 2）增加变量的值；3）把local的值写回，让其它的线程可见。这3步的jvm指令为：

```
mov    0xc(%r10),%r8d ; Load
inc    %r8d           ; Increment
mov    %r8d,0xc(%r10) ; Store
lock addl $0x0,(%rsp) ; StoreLoad Barrier
```

- 内存屏障（[memory barrier](http://en.wikipedia.org/wiki/Memory_barrier)）是一个CPU指令。基本上，它是这样一条指令： a) 确保一些特定操作执行的顺序； b) 影响一些数据的可见性(可能是某些指令执行后的结果)。编译器和CPU可以在保证输出结果一样的情况下对指令重排序，使性能得到优化。插入一个内存屏障，相当于告诉CPU和编译器先于这个命令的必须先执行，后于这个命令的必须后执行。内存屏障另一个作用是强制更新一次不同CPU的缓存。例如，一个写屏障会把这个屏障前写入的数据刷新到缓存，这样任何试图读取该数据的线程将得到最新值，而不用考虑到底是被哪个cpu核心或者哪颗CPU执行的。

### volatile为什么没有原子性?

从Load到store到内存屏障，一共4步，其中最后一步jvm让这个最新的变量的值在所有线程可见，也就是最后一步让所有的CPU内核都获得了最新的值，但**中间的几步（从Load到Store）**是不安全的，中间如果其他的CPU修改了值将会丢失。

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

