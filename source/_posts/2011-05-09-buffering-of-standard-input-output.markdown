---
author: admin
comments: true
date: 2011-05-09 14:30:39+00:00
layout: post
slug: buffering-of-standard-input-output
title: 标准输入／输出流的缓冲
wordpress_id: 397
categories:
- 默认
tags:
- buffering
- standard I/O
---

本文内容大部分内容来自《[Advanced Programming in the UNIX Environment](http://book.douban.com/subject/1788421/)》第5章第4节 Buffering。

题目：


> 下面的代码输出结果是什么？

>     
>     class A
>     {
>         private:
>             int m_value;
>         public:
>             A(int value)
>             {
>                 m_value = value;
>             }
>             void Print1()
>             {
>                 printf("hello world");
>             }
>             void Print2()
>             {
>                 printf("%d", m_value);
>             }
>     };
>     
>     int main(void)
>     {
>         A* pA = NULL;
>         pA->Print1();
>         pA->Print2();
>     
>         return 0;
>     }
> 
> 



显然应该是，能正常执行`Print1()`而不能执行`Print2()`，也就是说，在程序崩溃之前至少应该能输出`“hello world”`。原因看[这里](http://blog.developnotes.info/2009/12/22/0912222006/)

等一等，你可能会拷贝下来去编译执行一次，如果跟我一样在Mac下的话，你可能会发现不对劲的地方－为什么看不到`Print1()`的输出呢？为了确认答案，拷贝题目代码执行的结果中没有Print1()的输出，直接看到`Print2()`引用非法地址导致的错误：

    
    Segmentation fault


那这是什么原因呢？

`printf`函数定义：

    
    int  printf(const char * __restrict, ...)


它属于标准I/O库，标准I/O库提供了对输入输出的缓存以达到调用read和write次数最少的目的。嗯哼，“缓冲”是关键词，猜测Print1()的输出应该是被缓冲了吧。可具体是怎么缓冲的呢？

标准I/O库通过提供缓冲区，减少对于无缓冲的read和write方法的调用，增加效率。另一方面，缓冲区的存在也使得应用程序不用自己考虑最佳的读写块大小。不过，如上所见，缓冲区既是标准I/O库的便利之处，同时也是容易引起混乱的地方。标准I/O库提供三种缓冲方式：



	
  1. 完全缓冲，此种情形下，所有的I/O操作都在缓冲区被填满后才真正的发生；

	
  2. 行缓冲，只有当碰到换行符时才发生真正的I/O操作；

	
  3. 无缓冲，顾名思义，标准I/O库对I/O操作不做缓存。


ISO C对standard input、standard output以及standard error有以下规定：

	
  * 当且仅当不指向交互设备时，standard input和standard output是完全缓冲的

	
  * Standard error不允许完全缓冲


这个模糊不清的标准根本没说实现的时候具体应该怎么做，当standard input指向交互设备时，它应该无缓冲还是行缓冲？Standard error不允许完全缓冲，那么实现的时候采用行缓冲还是无缓冲呢？大多数的实现遵循以下原则：

	
  * Standard error总是实现为无缓冲流；

	
  * 除了standard error之外的其他流，倘若指向终端设备，则采取行缓冲。否则全部采用完全缓冲；


对默认的缓冲方式不满意的话，可以通过

    
    void setbuf(FILE * __restrict, char * __restrict);
    int  setvbuf(FILE * __restrict, char * __restrict, int, size_t);


两个函数更改缓冲区大小以及类型。

好了，既然知道内幕，现在修改main函数，把缓冲区设置成空试试

    
    int main(void)
    {
        setbuf(stdout, NULL);
    
        A* pA = NULL;
        pA->Print1();
        pA->Print2();
    
        return 0;
    }


再执行，这次`Print1()`的输出能够正常显示。
