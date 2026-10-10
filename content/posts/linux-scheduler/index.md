+++
date = 2026-04-11
title = "Linux中O(1)进程调度相关数据结构"
description = ""
slug = ""
authors = []
tags = ["调度器", "进程调度", "优先级", "O(1)调度"]
series = ["Linux系统编程"]
featuredImage = "assets/cover.png"
toc = true
+++


> 原创 于 2026-04-11 07:30:00 发布 · 公开 · 394 阅读 · 8 · 7 · 本内容遵循CC 4.0 BY-SA版权协议 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。 GEO检测 · 编辑
> 文章链接：https://blog.csdn.net/2401_87889177/article/details/159997859


说明：本文提到代码出自于Linux2.26.18。源码下载地址 [Linux2.26.x](https://www.kernel.org/pub/linux/kernel/v2.6/) 

## 一、标题内嵌链表的应用

#### 已知成员和类型求地址

现有一个结构体

```c
struct A
{
	int a;
	int b;
	double c;
};
```

如果我们知道了c的地址和A类型，就能通过以下代码知道c成员和整个结构体起始位置的偏移。

```c
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER) 
```

offsetof位于stddef.h头文件中。

原理：能够通过指针访问结构体的成员，假设0地址出有一个TYPE类型的结构体，将0强转成TYPE*类型的指针然后通过箭头访问到了MEMBER。然后MEMBER的地址减去起始地址0就是偏移了。
示例

```c
struct A
{
	int a;
	int b;
	double c;
};

int main()
{
    
    struct A abc = {1,2,3.5};
    size_t offset = offsetof(struct A, c);
    printf("c的偏移为%zu, 整个结构体的地址是%p, b的值为%d\n",
	    offset, &abc.c, 
	    ((struct A*)((char*)&abc.c - offset))->b);
    return 0;
}
```

输出

```shell
c的偏移为8, 整个结构体的地址是0x7ffd3b75ec18, b的值为2
```

#### 内嵌链表作用

```c
struct list_head {
	struct list_head *next, *prev;
};
```

这是Linux中的双链表，它没有数据域，只有两个指针。
 ![双链表](./assets/01_1.png)

如果一个结构体中有多个这样的链表，那么，这个结构体就能够同时被链入多个链表中。提高了链表的可扩展性。

## 二、task_struct细节

Linux中的PCB叫做 `task_struct` 。有记录进程相关信息，保存上下文等作用。Linux就是通过内嵌双链表将一个task_struct对象链入多个数据结构中，例如

```c
struct task_struct {
	...
	struct list_head run_list;
	...
	struct list_head tasks;
	...
};
```

task_struct中有多个 `struct list_head` 的双链表结点，这里以 `run_list` 和 `tasks` 为例。run_list的功能是将该进程链进运行队列，tasks的作用是通过该链表结点将进程链入全局进程链表中。

## 三、rq运行队列

#### 总览进程调度中的数据结构

<img src="./assets/01_2.jpeg" alt="在这里插入图片描述" style="max-width:500px; box-sizing:content-box;" />

```c
struct rq {
	...
	struct prio_array *active, *expired, arrays[2];
	...
};
```

运行队列rq由两个 `struct prio_array` 组成，而 `active` 和 `expired` 指向arrays [0](%E6%B4%BB%E8%B7%83%E8%BF%9B%E7%A8%8B) 或者arrays [1](%E8%BF%87%E6%9C%9F%E8%BF%9B%E7%A8%8B) ，array是两个具体的优先级调度队列。
structprio_array定义如下

```c
struct prio_array {
	unsigned int nr_active;
	DECLARE_BITMAP(bitmap, MAX_PRIO+1); /* include 1 bit for delimiter */
	struct list_head queue[MAX_PRIO];
};
```

queue[140]是一由140个链表组成的优先级队列。通过进程优先级（0~139）映射到这个数组中，有哈希表的思想。

![queue140](assets/queue140.png)

下标从100到139是普通进程链表，即我们平时看见的进程，0~99是实时进程链表，用于实时操作系统，Linux不是一个存粹的分时操作系统或实时操作系统。实时操作系统在特殊场景下才会使用，所以我们看见的基本上都是普通进程。

CPU依次从数字小的下标开始调度链表中的头节点，时间片耗尽过后，将该进程PCB移入expired的调度队列。直到140个链表中的进程全部调度完成。然后交换active和expired的指针，开始调度active指向的队列。新进程也是插入的expired队列中。

#### 位图优化

CPU每次调度active队列，就要找到第一个优先级低且非空的链表。如果遍历140个元素的数组，虽然时间复杂度也是O(1)，但Linux并没有这样做，而是采用位图优化。

```c
	DECLARE_BITMAP(bitmap, MAX_PRIO+1);
```

这里声明了一个位图（如果对应优先级的下标处链表非空，则位图存储为1）由6个unsigned long组成共160位，能够覆盖140个链表头，一个unsigned long由32比特位组成，那么就能够一次性判断32个比特位中是否有不为空的链表，然后就能够快速锁定区域，进程调度。

## 总结

在active指向的优先级队列中，通过位图在140个数组中找到优先级低的非空链表，时间复杂度为O(1)。然后不断调度链表头节点，删除active中进程，添加到expired中，时间复杂度为O(1)。active调度完了，交换active和expired指向内容O(1)，然后重复。
总体时间复杂度为O(1)。