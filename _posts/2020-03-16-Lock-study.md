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



​	

| 锁状态   | 25bit                                          | 4bit         | 1bit是否是偏向锁 |
| -------- | ---------------------------------------------- | ------------ | ---------------- |
| GC标记   | 空                                             |              |                  |
| 重量级锁 | 指向重量级锁Monitor指针                        |              |                  |
| 轻量级锁 | 指向线程栈中锁记录的指针pointer to lock Record |              |                  |
| 偏向锁   | 线程ID、 Epoch                                 | 对象分代年龄 | 1                |
| 无锁     | 对象的hashCode                                 | 对象分代年龄 | 0                |



# Lock

## ReetrantLock

- AQS
  - AQS是Abstract Queued Synchronizer，它是用来构建锁或其他同步组件的基础框架，内部通过一个int类型的成员变量state来控制同步状态,当state=0时，则说明没有任何线程占有共享资源的锁，当state=1时，则说明有线程目前正在使用共享变量，其他线程必须加入同步队列进行等待
  - AQS内部通过内部类Node构成FIFO的同步队列来完成线程获取锁的排队工作，同时利用内部类ConditionObject构建等待队列，当Condition调用wait()方法后，线程将会加入等待队列中，而当Condition调用signal()方法后，线程将从等待队列转移动同步队列中进行锁竞争。
  - 涉及到两种队列，一种的同步队列，当线程请求锁而等待的后将加入同步队列等待，而另一种则是等待队列(可有多个)，通过Condition调用await()方法释放锁后，将加入等待队列。

# Synchronized

- 操作系统提供monitor机制，c++实现，mutex 机制- > pthread_mutex_lock (内核态)

- Object的wait/notify/notifyAll方法都依赖于ObjectMonitor 
- Java对象存储在内存中，包含对象头、实例数据和对齐填充三部分，其中对象头包含锁标志位
- 翻译成指令是monitorenter 、monitorexit
- 

# 偏向锁

- 偏向锁的目标是，减少无竞争且只有一个线程使用锁的情况下，使用轻量级锁产生的性能消耗。轻量级锁每次申请、释放锁都至少需要一次CAS，但偏向锁只有初始化时需要一次CAS。

- “偏向”的意思是，*偏向锁假定将来只有第一个申请锁的线程会使用锁*（不会有任何线程再来申请锁），因此，*只需要在Mark Word中CAS记录owner（本质上也是更新，但初始值为空），如果记录成功，则偏向锁获取成功*，记录锁状态为偏向锁，*以后当前线程等于owner就可以零成本的直接获得锁；否则，说明有其他线程竞争，膨胀为轻量级锁*。

  