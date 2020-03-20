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

~还待进一步完善~



# 海量数据处理

# 海量数据处理

采用巧妙的算法搭配合适的数据结构，如Bloom filter/Hash/bit-map/堆/数据库或倒排索引/trie树。

**1、海量日志数据，提取出某日访问百度次数最多的那个IP**

​	1）分而治之/hash映射：针对数据太大，内存受限，只能是：把大文件化成(取模映射)小文件，即16字方针：大                  而化小，各个击破，缩小规模，逐个解决
​    2) hash_map统计：当大文件转化了小文件，那么我们便可以采用常规的hash_map(ip，value)来进行频率统计。
​    3) 堆/快速排序：统计完了之后，便进行排序(可采取堆排序)，得到次数最多的IP。

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
$ cat /etc/passwd | awk 'BEGIN {FS=":"} $3 < 10 {print $1 "\t " $3}'
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

- 确保最后一个确认报文能后到达。如果B没有收到A发送的确认报文，那么就会重新发送连接释放请求报文，A等待一段时间就是为了处理这种情况的发送。
- 等待一段时间是为了让本连接持续时间内所产生的对所有报文都从网络上消失，使得下一个新的连接不会出现旧的连接请求报文。