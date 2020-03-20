---
layout:     post
title:      "Java8相关知识"
subtitle:   "Java8的一些新特性"
date:       2020-03-11
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - Java
---

~还待进一步完善~

### HashMap的优化

- JDK7基于链表+数组实现，底层维护一个Entry数组。
  - 发生hash冲突时，新元素插入到链表头中，即新元素总是添加到数组中，就元素移动到链表中。 
  - 在扩容resize（）过程中，采用单链表的头插入方式，在将旧数组上的数据 转移到 新数组上时，按旧链表的正序遍历链表、在新链表的头部依次插入，即在转移数据、扩容后，容易出现链表逆序的情况 。 多线程下resize()容易出现死循环。
- JDK8改成了基于位桶+链表/红黑树的方式实现,底层维护一个Node数组。
  - 发生hash冲突时，会优先判断该节点的数据结构式是红黑树还是链表，如果是红黑树，则在红黑树中插入数据；如果是链表，则将数据插入到链表的尾部并判断链表长度是否大于8，如果大于8要转成红黑树。
  - 扩容时，按旧链表的正序遍历链表、在新链表的尾部依次插入，所以不会出现链表 逆序、倒置的情况，故不容易出现环形链表的情况。

## Lambda表达式





## Stream流



```java
	List<String> a = new ArrayList<>();
	a.stream().forEach(c-> System.out.println(c));
```

Stream中还有fifter、sorted、Match、map、Reduce这一类的API

### 构造函数和方法引用





## Default Method





