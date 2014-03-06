---
author: admin
comments: true
date: 2010-03-19 07:39:51+00:00
layout: post
slug: keyword-restrict
title: restrict关键字
wordpress_id: 268
categories:
- 默认
tags:
- restrict
- 细节
---

看代码看到个restrict，头一次知道还有这么个关键字，在[维基](http://en.wikipedia.org/wiki/Restrict)查到基本用法和解释。本着好记性不如烂笔头的原则，翻译[这篇](http://en.wikipedia.org/wiki/Restrict)维基贴在这里。

在C语言的一个标准(C99)里，有一个用于修饰指针声明的关键字－`restrict`，它是程序员给编译器的一个意向声明，表明只有此指针或者基于此指针的某个值(比如 pointer+1)可以访问其指向的对象。这个关键字限制了指针别名(译注：即指向同一对象的不同指针)，目的在于进行编译器优化。如果这个程序员没有遵守这个意向声明，并且这个对象由一个独立指针访问到的话，会造成未定义的行为。( 译注：我也没搞懂这一大段话什么意思 ^_^ 不过看下面的例子应该差不多能理解这个关键字的用法)

如果知道只有一个指针指向某个内存块的话，编译器就能进行优化，以生成更高效的代码。下面这个例子能清楚解释这一点。


    void updatePtrs(size_t *ptrA, size_t *ptrB, size_t *val)
    {
        *ptrA += *val;
        *ptrB += *val;
    }

这段代码里面，指针`ptrA` , `ptrB`, `val`可能指向同一块内存，所以编译器不能对代码进行优化。


    load R1 ← *val  ; 加载val指针的
    load R2 ← *ptrA ; 加载ptrA指针的值&lt;/pre&gt;
    add  R2 += R1   ; 相加
    set  R2 → *ptrA ; 更新ptrA指针的值
    ; 对ptrB指针的操作类似，注意，这里val指针被加载两次，因为ptrA有可能和val指向同一块内存
    load R1 ← *val
    load R2 ← *ptrB
    add  R2 += R1
    set  R2 → *ptrB

如果使用restrict关键字，函数声明为：

    void updatePtrs(size_t *restrict ptrA, size_t *restrict ptrB, size_t *restrict val);

编译器就能知道ptrA, ptrB, val三个指针指向不同的内存块，更新其中一个不会影响到其他的，从而对代码进行优化。当然，这里要由程序员保证这三个指针不指向同一块内存。

    load R1 ← *val
    load R2 ← *ptrA
    add  R2 += R1
    set  R2 → *ptrA
    ;编译器知道指针val的值没有改变，所以没有重新加载它，优化了汇编代码
    load R2 ← *ptrB
    add  R2 += R1
    set  R2 → *ptrB

还有一个例子是`memcpy`函数，`memcpy(void*, void*, nbytes)`的前两个指针参数被声明为`restrict`，告诉编译器两个数据块没有重叠，便于进行优化，使得`memcpy`的速度更快。如果程序员用相同的指针作为这两个参数的值，那么memcpy函数会产生未定义的行为。
