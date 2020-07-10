---
layout:     post
title:      "Nio学习"
subtitle:   "Nio的基础知识"
date:       2020-05-08
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - Nio
	- Java
---

## I/O模型

- BIO : 同步阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销；
- NIO：同步非阻塞，面向缓冲区，面向块编程。服务器实现模式为一个线程处理多个请求，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求就进行处理；
- AIO： 异步非阻塞，引入异步通道，采用了Proactor模式，简化了程序的编写，有效的请求才启动线程，它的特点是现有操作系统完成后通知服务端程序启动线程来处理，一般用户连接数较多时间较长的应用。

## NIO的三大核心

- 每个channel都会对应一个Buffer
- Selector对应一个线程，一个线程对应多个channel
- channel注册到Selector中，程序切换到哪个channel是由事件决定的，Event
- Selector会根据不同的事件，在各个channel上切换
- Buffer就是一个内存块，底层是有一个数组
- 数据的读取写入是通过Buffer，flip切换读写，channel是双向的。

### Selector

Selector能够检测多个注册的通道上是否有事件发生，如果有事件发生，便获取事件然后针对每个事件进行相应的处理。

### Channel

- Channel类似于Stream，但是可以同时进行读写，而Stream只能读或者只能写
- Channel可以实现异步读取数据
- Channel可以从Buffer中读写数据

### Buffer

Buffer本质上是一个可以读写数据的内存块，可以理解成一个容器对象，该对象提供了一组方法，可以更轻松地使用内存块，Buffer对象内置了一些机制，能够跟踪和记录Buffer的状态变化情况。Channel提供从文件、网络读取数据的渠道，但是读取或写入的数据都必须经由Buffer。

## NIO和零拷贝

Java中常用的零拷贝有mmap和sendFile。零拷贝是从OS角度来说的，因为内核缓冲区之间，没有数据是重复的。（只有kernel buffer一份数据）。零拷贝还减少了环境切换，带来了性能优势。

### mmap

mmap通过内存映射，将文件映射到内核缓冲区，同时用户空间可以共享内核空间的数据。这样，在进行网络传输时，就可以减少内核空间到用户空间的拷贝次数。

### sendFile

kernel Buffer - > protocol engine , 中间没有经过socket buffer。实现CPU零拷贝

## Netty优点

- 高性能、高吞吐、延迟低，资源消耗更少

