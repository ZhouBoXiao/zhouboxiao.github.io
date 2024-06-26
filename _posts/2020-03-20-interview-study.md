---
layout:     post
title:      "面试相关"
subtitle:   "面试相关的一些问题"
date:       2020-03-20
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - 面试
---

还待进一步完善

# Jdk 1.8的HashMap中resize中的2次幂的扩展

- 通过把长度扩展为原来的两倍，元素不用重新计算hash，只需原有的hash&(n-1)，
- 原来的hash新增的那个bit是1还是0，是0的话索引没变，是1的话索引变成”原索引+oldCap“

# Nagle算法

传送一个只拥有1个字节有效数据的数据包，却要发费40个字节长包头（即ip头20字节+tcp头20字节）的额外开销，这种有效载荷（payload）利用率极其低下的情况被统称之为愚蠢窗口症候群（Silly Window Syndrome）。可以看到，这种情况对于轻负载的网络来说，可能还可以接受，但是对于重负载的网络而言，就极有可能承载不了而轻易的发生拥塞瘫痪。

针对上面提到的情况，Nagle算法的改进在于：如果发送端欲发送多次包含少量字符的数据包，则发送端会先将第一个小包（小于MSS 1460）发送出去，而将后面到达的少量字符数据都缓存起来而不立即发送，直到收到接收端对前一个数据包报文段的ACK确认、或当前字符属于紧急数据，或者积攒到了一定数量的数据（比如缓存的字符数据已经达到数据包报文段的最大长度）等多种情况才将其组成一个较大的数据包发送出去。

延迟ACK：

如果tcp对每个数据包都发送一个ack确认，那么只是一个单独的数据包为了发送一个ack代价比较高，所以tcp会延迟一段时间，如果这段时间内有数据发送到对端，则捎带发送ack，如果在延迟ack定时器触发时候，发现ack尚未发送，则立即单独发送；

# 线程的通信方式

1. 等待通知机制。两个线程通过对同一对象调用等待 wait() 和通知 notify() 方法来进行通讯。
2. 信号量机制
3. 锁机制
4. join() 方法 （内部实现为wait）
5. volatile共享内存
6. CountDownLatch
7. CyclicBarrier
8. 线程响应中断
9. 线程池 awaitTermination() 方法
10. 管道通信

# HashTable和HashMap区别

- 继承的父类不同
- 线程安全性不同
- 内部实现使用的数组初始化和扩容方式不同
- hash值不同

# BIO、NIO、AIO概念和区别

- BIO （Blocking I/O）：同步阻塞I/O模式，数据的读取写入必须阻塞在一个线程内等待其完成。
- NIO是一种同步非阻塞的I/O模型，在Java 1.4 中引入了 NIO 框架，对应 java.nio 包，提供了 Channel , Selector，Buffer等抽象。单线程中从通道读取数据到Buffer，同时可以继续做别的事情，当数据读取到Buffer中后，线程再继续处理数据。写数据也是一样的。
- 异步非阻塞的IO模型。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

# 如何避免线程死锁

1. **破坏互斥条件** ：这个条件我们没有办法破坏，因为我们用锁本来就是想让他们互斥的（临界资源需要互斥访问）。
2. **破坏请求与保持条件** ：一次性申请所有的资源。
3. **破坏不剥夺条件** ：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。
4. **破坏循环等待条件** ：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。

# Cookie和Session的的区别

Cookie 和 Session都是用来跟踪浏览器用户身份的会话方式，但是两者的应用场景不太一样。

**Cookie 一般用来保存用户信息** 比如①我们在 Cookie 中保存已经登录过得用户信息，下次访问网站的时候页面可以自动帮你登录的一些基本信息给填了；②一般的网站都会有保持登录也就是说下次你再访问网站的时候就不需要重新登录了，这是因为用户登录的时候我们可以存放了一个 Token 在 Cookie 中，下次登录的时候只需要根据 Token 值来查找用户即可(为了安全考虑，重新登录一般要将 Token 重写)；③登录一次网站后访问网站其他页面不需要重新登录。**Session 的主要作用就是通过服务端记录用户的状态。** 典型的场景是购物车，当你要添加商品到购物车的时候，系统不知道是哪个用户操作的，因为 HTTP 协议是无状态的。服务端给特定的用户创建特定的 Session 之后就可以标识这个用户并且跟踪这个用户了。

Cookie 数据保存在客户端(浏览器端)，Session 数据保存在服务器端。

Cookie 存储在客户端中，而Session存储在服务器上，相对来说 Session 安全性更高。如果使用 Cookie 的一些敏感信息不要写入 Cookie 中，最好能将 Cookie 信息加密然后使用到的时候再去服务器端解密。

# String

String 类中使用 final 关键字修饰字符数组来保存字符串，`private final char value[]`，所以 String 对象是不可变的。

在 Java 9 之后，String 类的实现改用 byte 数组存储字符串 `private final byte[] value`;

# 双亲委派机制

双亲委派模型工作过程是：如果一个类加载器收到类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器完成。每个类加载器都是如此，只有当父加载器在自己的搜索范围内找不到指定的类时，子加载器才会尝试自己去加载。

- 如何打破？

  - 默认的loadClass方法是实现了双亲委派机制的逻辑，即会先让父类加载器加载，当无法加载时才由自己加载。为了破坏双亲委派机制必须重写loadClass方法，即这里先尝试交由System类加载器加载，加载失败才会由自己加载。它并没有优先交给父类加载器，这就打破了双亲委派机制。

- 打破的案例： Tomcat
  - 为了Web应用程序类之间隔离，为每个应用程序创建WebAppClassLoader类加载器
  - 为了Web应用程序类之间共享，把ShareClassLoader作为WebAppClassLoader的父加载器，如果WebAppClassLoader加载不到，则尝试ShareClassLoader
  - 为了Tomcat本身与Web应用程序类隔离，用CatalinaClassLoader类加载器进行隔离，CatalinaClassLoader加载Tomcat本身的类
  - 为了Tomcat与Web应用程序类共享，用CommonClassLoader作为CatalinaClassLoader和ShareClassLoader的父类加载器
  - ShareClassLoader、CatalinaClassLoader、CommonClassLoader的目录可以在Tomcat的catalina.properties进行配置
    

# Tomcat对ThreadPoolExecutor线程池的扩展

核心：重写了父类的execute方法，进行扩展

```java
  
public void execute(Runnable command) {
	execute(command,0,TimeUnit.MILLISECONDS);
}
 
public void execute(Runnable command, long timeout, TimeUnit unit) {
    // 将自己的计算器叠加
	submittedCount.incrementAndGet();
	try { // 调用父类的模板流程
		super.execute(command);
	} catch (RejectedExecutionException rx) {
        // 父类执行拒绝策略（肯定是到达了最大线程数） 
		if (super.getQueue() instanceof TaskQueue) {
			final TaskQueue queue = (TaskQueue)super.getQueue();
			try {
                // 尝试着真正将任务添加到队列中，没有成功则说明队列也真的满了，该真的执行拒绝策略
				if (!queue.force(command, timeout, unit)) {
					submittedCount.decrementAndGet();
					throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
				}
			} catch (InterruptedException x) {
				submittedCount.decrementAndGet();
				throw new RejectedExecutionException(x);
			}
		} else {
			submittedCount.decrementAndGet();
			throw rx;
		}
 
	}
}
```

  1、将自己的计算器叠加

  2、调用父类的模板方法执行

  3、父类执行拒绝策略，说明已经到了核心线程数。那么获取父类的队列，尝试着将任务添加到队列中（**这样就实现了先到创建最大线程，再放入队列中**），如果队列也满了放不进去，再执行真正的拒绝策略。

# ORM

对象-关系映射，主要实现程序对象到关系数据库数据的映射。

程序员不需要编写复杂的SQL语句，直接操作Java对象即可，从而大大降低了代码量，也使程序员更加专注于业务逻辑的实现。

- 好处
  - 提高开发效率，降低开发成本 ，使开发更加对象化
  - 可移植

# Spring IOC

即把各个对象类封装之后，通过IoC容器来关联这些对象类。这样对象与对象之间就通过IoC容器进行联系，但对象与对象之间并没有什么直接联系。

- 依赖注入（DI）

  - 由IoC容器在运行期间，动态地将某种依赖关系注入到对象之中。
  - 依赖注入（DI）和控制反转（IoC）是从不同的角度描述的同一件事情，就是指通过引入IoC容器，利用依赖关系注入的方式，实现对象之间的解耦

- 好处

  - 可维护性比较好，非常便于进行单元测试，便于调试程序和诊断故障。
  - 可复用性好

- 工厂模式提供创建对象的接口

- AbstractBeanFactory#doGetBean

  - 解析别名

  - 尝试从缓存中获取（singletonObjects）

  - 判断有没有父工厂

  - bean定义的抽象校验

  - dependsOn的优先级校验

  - createBean()

    


# MySQL中varchar最大长度是多少？

- varchar最多能存储65535个字节的数据。varchar 的最大长度受限于最大行长度（max row size，65535bytes）。65535并不是一个很精确的上限，可以继续缩小这个上限。65535个字节包括所有字段的长度，变长字段的长度标识（每个变长字段额外使用1或者2个字节记录实际数据长度）、NULL标识位的累计。
- **NULL标识位，如果varchar字段定义中带有default null允许列空,则需要需要1bit来标识，每8个bits的标识组成一个字段。一张表中存在N个varchar字段，那么需要（N+7）/8 （取整）bytes存储所有的NULL标识位。**如果数据表只有一个varchar字段且该字段DEFAULT NULL，那么该varchar字段的最大长度为65532个字节，即65535-2-1=65532 byte。

# EPOLL ET和LT的区别

- LT：水平触发
  - 如果事件来了，不管来了几个，只要仍然有未处理的事件，epoll都会通知你。比如事件来了，打印一行通知，但是不去处理事件，那么会不停滴打印通知。水平触发模式的 epoll 的扩展性很差。
- ET：边缘触发
  - 如果事件来了，不管来了几个，你若不处理或者没有处理完，除非下一个事件到来，否则epoll将不会再通知你。 比如事件来了，打印一行通知，但是不去处理事件，那么打印一行通知

# OCP开发-封闭原则

**OCP概述**
　　遵循开放-封闭原则设计出的模块具有两个主要的特征。它们是：

1. **对于扩展是开放的（open for extension）。**这意味着模块的行为是可以扩展的。当应用的需求改变时，我们可以对模块进行扩展，使其具有满足那些改变的新行为。换句话说，我们可以改变模块的功能。
2. **对于修改是封闭的（closed for modification）。**对模块行为进行扩展时，不必改动模块的源代码或者二进制代码。模块的二进制可执行版本，无论是可链接的库、DLL或者.EXE文件，都无需改动。

# Java线程和计算机的核怎么对应的

具体到我们平时常用的JVM实现，Oracle/Sun的HotSpot VM，它是用1:1模型来实现Java线程的，也就是说一个Java线程是直接通过一个OS线程来实现的，中间并没有额外的间接结构。而且HotSpot VM自己也不干涉线程的调度，全权交给底下的OS去处理。

## Java 的排序算法实现

Java 主要排序方法为 java.util.Arrays.sort()，对于原始数据类型使用**三向切分的快速排序**，对于引用类型使用**归并排序**。

# 红黑树的优点

- 红黑树RB-Tree能够以**O(log2 n)** 的时间复杂度进行搜索、插入、删除操作
- 如果插入一个node引起了树的不平衡，AVL和RB-Tree都是最多只需要**2次**旋转操作，即两者都是O(1)
- 但是在删除node引起树的不平衡时，最坏情况下，AVL需要维护从被删node到root这条路径上所有node的平衡性，因此需要旋转的量级O(logN)，而RB-Tree最多只需**3次**旋转，只需要O(1)的复杂度。

# 海量数据处理

采用巧妙的算法搭配合适的数据结构，如Bloom filter/Hash/bit-map/堆/数据库或倒排索引/trie树。

**1、海量日志数据，提取出某日访问百度次数最多的那个IP**

​	1）分而治之/hash映射：针对数据太大，内存受限，只能是：把大文件化成(取模映射)小文件，即16字方针：大                  而化小，各个击破，缩小规模，逐个解决
​    2) hash_map统计：当大文件转化了小文件，那么我们便可以采用常规的hash_map(ip，value)来进行频率统计。
​    3) 堆/快速排序：统计完了之后，便进行排序(可采取堆排序)，得到次数最多的IP。

**2、100亿个整数，找出中位数**

首先我们将int划分为2^16个区域，然后读取数据统计落到各个区域里的数的个数，之后我们根据统计结果就可以判断中位数落到那个区域，同时知道这个区域中的第几大数刚好是中位数。然后第二次扫描我们只统计落在这个区域中的那些数就可以了。



# HTTP状态码

| 状态码 | 类别                             | 含义                       |
| ------ | -------------------------------- | -------------------------- |
| 1XX    | Informational（信息性状态码）    | 接收的请求正在处理         |
| 2XX    | Success（成功状态码）            | 请求正常处理完毕           |
| 3XX    | Redirection（重定向状态码）      | 需要进行附加操作以完成请求 |
| 4XX    | Client Error（客户端错误状态码） | 服务器无法处理请求         |
| 5XX    | Server Error（服务器错误状态码） | 服务器处理请求出错         |

# left join和right join和全连接区别?

- left join(左联接) 返回包括左表中的所有记录和右表中联结字段相等的记录

- right join(右联接) 返回包括右表中的所有记录和左表中联结字段相等的记录
- inner join(内连接) 只返回两个表中联结字段相等的行

- full join or full outer join能返回左右表里的所有记录，其中左右表里能关联起来的记录被连接后返回。

- CROSS JOIN 返回左表与右表之间符合条件的记录的迪卡尔集。

- SELF JOIN 返回表与自己连接后符合条件的记录，一般用在表里有一个字段是用主键作为外键的情况。 

# 进程间通信方式

进程间通信（IPC，InterProcess Communication）的主要方式包括：管道、FIFO、消息队列、信号量、共享内存和socket。

- 信号
  
  - 信号是一种比较复杂的通信方式,用于通知接收进程某个事件已经发生。
  
- 管道

  - 无名管道，半双工，管道只能在具有公共祖先的两个进程之间使用，通常，一个管道由一个进程创建，在进程调用fork之后，这个管道就能在父进程和子进程之间使用了。

  - ```
    #include<unistd.h>
    int pipe(int fd[2]);      //成功返回0，出错返回-1
    //经由参数fd返回两个文件描述符：fd[0]为读而打开，fd[1]为写而打开。
    ```

- FIFO

  - 命名管道，可以在不同的程序之间交换数据。

  - 其实是一种文件类型，创建FIFO类似创建文件：

    ```
    #include<sys/stat.h>
    int mkfifo(const char *path, mode_t mode);
    
    int mkfifoat(int fd, const char *path, mode_t mode);    //成功返回0，失败返回-1
    ```

  - shell命令使用FIFO将数据从一条管道传送到另一条管道时，无须创建中间的临时文件。

  - 客户进程-服务器进程应用程序中，FIFO用作**汇聚点**，在客户进程和服务器进程二者之间传递数据。

- 消息队列

  - 消息队列是消息的链接表，存储在内核中，有消息队列标识符标识。

  - UNIX允许不同进程将格式化的数据流以消息队列形式发送给任意进程。

  - ```
    #include<sys/msg.h>
    //创建一个新的消息队列或打开一个现有队列
    int msgget(key_t key, int flag);
    //将新消息添加到队列尾端
    int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);  
    //从队列中去消息
    ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag);
    ```

- 信号量

  - 一个计数器，用于为多个进程提供共享数据对象的访问。
  - 不是用于交换大批数据,而用于多线程之间的同步，它常作为一种锁机制，防止某进程在访问资源时其它进程也访问该资源。
  - 为了获得共享资源，进程需要执行下列操作：
    - 测试控制该资源的信号量；
    - 若此信号量的值为正，则进程可以使用该资源。在这种情况下，进程会将信号量值减1，表示它使用了一个资源单位；
    - 否则，若此信号量的值为0，则进程进入**休眠状态**，知道信号量的值大于0。进程被唤醒后，返回步骤1）。

- 共享存储

  - 共享存储允许两个或多个进程共享一个给定的存储区。**信号量**用于同步共享存储访问。

- socket

  - socket是在应用层和传输层之间的一个抽象层，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用，从而实现进程在网络中通信。



# TCP和UDP的特点

- TCP（Transmission Control Protocol）是面向连接的，提供可靠交付，有流量控制和拥塞控制，提供全双工通信，面向字节流，一对一通信
- UDP（User Datagram Protocol）是无连接的，尽最大可能交付，没有拥塞控制，面向报文，支持一对一和一对多或者多对多的通信。

## awk

是由 Alfred Aho，Peter Weinberger 和 Brian Kernighan 创造，awk 这个名字就是这三个创始人名字的首字母。

awk 每次处理一行，处理的最小单位是字段，每个字段的命名方式为：$n，n 为字段号，从 1 开始，$0 表示一整行。

示例：取出最近五个登录用户的用户名和 IP。首先用 last -n 5 取出用最近五个登录用户的所有信息，可以看到用户名和 IP 分别在第 1 列和第 3 列，我们用 $1 和 $3 就能取出这两个字段，然后用 print 进行打印。

```
$ last -n 5
dmtsai pts/0 192.168.1.100 Tue Jul 14 17:32 still logged in
dmtsai pts/0 192.168.1.100 Thu Jul 9 23:36 - 02:58 (03:22)
dmtsai pts/0 192.168.1.100 Thu Jul 9 17:23 - 23:36 (06:12)
dmtsai pts/0 192.168.1.100 Thu Jul 9 08:02 - 08:17 (00:14)
dmtsai tty1 Fri May 29 11:55 - 12:11 (00:15)
$ last -n 5 | awk '{print $1 "\t" $3}'
```

可以根据字段的某些条件进行匹配，例如匹配字段小于某个值的那一行数据。

```
$ awk '条件类型 1 {动作 1} 条件类型 2 {动作 2} ...' filename
```

示例：/etc/passwd 文件第三个字段为 UID，对 UID 小于 10 的数据进行处理。

```
$ cat /etc/passwd | awk 'BEGIN {FS=":"} $3 < 10  {print $1 "\t " $3}'
root 0
bin 1
daemon 2
```

awk 变量：

| 变量名称 | 代表意义                     |
| -------- | ---------------------------- |
| NF       | 每一行拥有的字段总数         |
| NR       | 目前所处理的是第几行数据     |
| FS       | 目前的分隔字符，默认是空格键 |

示例：显示正在处理的行号以及每一行有多少字段

```
$ last -n 5 | awk '{print $1 "\t lines: " NR "\t columns: " NF}'
dmtsai lines: 1 columns: 10
dmtsai lines: 2 columns: 10
dmtsai lines: 3 columns: 10
dmtsai lines: 4 columns: 10
dmtsai lines: 5 columns: 9
```

# TIME_WAIT

客户端接收到服务端的FIN报文后进入此状态，此时并不是直接进入CLOSED状态，还需要等待一个时间器设置的时间2MSL。理由有两个：

- 确保最后一个确认报文能够 到达。如果B没有收到A发送的确认报文，那么就会重新发送连接释放请求报文，A等待一段时间就是为了处理这种情况的发送。
- 等待一段时间是为了让本连接持续时间内所产生的对所有报文都从网络上消失，使得下一个新的连接不会出现旧的连接请求报文。

**Shell** **数组**元素个数${#array[@]} **数组**的所有元素${array[*]} 字符串**长度**${#str}