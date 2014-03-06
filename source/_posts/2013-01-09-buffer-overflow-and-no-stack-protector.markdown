---
author: admin
comments: true
date: 2013-01-09 12:49:51+00:00
layout: post
slug: buffer-overflow-and-no-stack-protector
title: 缓冲区溢出和-fno-stack-protector
wordpress_id: 498
categories:
- Security
tags:
- buffer overflow
- stack protector
---

[缓冲区溢出](http://en.wikipedia.org/wiki/Buffer_overflow)早在1972年就被发现，直到今天，还有很多程序员中招。用C语言不注意检查数组或者指针边界，一些C标准库里的函数使用不当，都有可能引起缓冲区溢出。

    #include <stdio.h>
    char *_gets(char *s)
    {
        int c;
        char *dest = s;
        int gotchar=0;
        while ((c = getchar()) != '\n' && c!= EOF) {
            *dest++ = c;
            gotchar = 1;
        }
    
        *dest++ = '\0';
        if (c == EOF && !gotchar)
            return NULL;
        return s;
    }
    
    void echo()
    {
        char buf[8];
        _gets(buf);
        puts(buf);
    }

这段代码用库函数`getchar()`实现另一个库函数`gets()`，一个存在缓冲区溢出漏洞的版本`_gets()`，因为它没有办法确定用来保存输入的空间是否足够。如果空间足够小，像`echo()`里的8个字节，那么任何长度超过7个字符的输入都会导致数组越界。

    _echo:
    00000000000000b0    pushq   %rbp
    00000000000000b1    movq    %rsp,%rbp
    00000000000000b4    subq    $0x10,%rsp
    00000000000000b8    leaq    __gets.eh(%rbp),%rax
    00000000000000bc    movq    %rax,%rcx
    00000000000000bf    movq    %rcx,%rdi
    00000000000000c2    movq    %rax,0xf0(%rbp)
    00000000000000c6    callq   __gets
    00000000000000cb    movq    0xf0(%rbp),%rax
    00000000000000cf    movq    %rax,%rdi
    00000000000000d2    callq   _puts
    00000000000000d7    addq    $0x10,%rsp
    00000000000000db    popq    %rbp
    00000000000000dc    ret

汇编代码（由i686-apple-darwin11-llvm-gcc-4.2 (GCC) 4.2.1产生）说明了栈的组织方式。

![](http://farm9.staticflickr.com/8073/8363553547_443c523657.jpg)

一旦越界的字符很多，就会破坏调用者中保存的状态，导致程序进入异常状态。

所谓的缓冲区溢出攻击，就是利用这种情况。输入特定的字符串给程序，内容是一些可执行代码的字节编码，让程序执行外部代码，或者覆盖返回地址，让程序跳转到攻击代码，然后，攻击者就可以为所欲为。

缓冲区溢出攻击太过普遍，以致于现代的编译器和操作系统已经实现了一些机制来最小化这种攻击。GCC会尝试加入额外的代码以检测栈是否溢出，选项-fstack-protector说明：


> -fstack-protector
> 
  Emit extra code to check for buffer overflows, such as stack smashing
  attacks. This is done by adding a guard variable to functions with
  vulnerable objects. This includes functions that call alloca, and
  functions with buffers larger than 8 bytes. The guards are
  initialized when a function is entered and then checked when the
  function exits. If a guard check fails, an error message is printed
  and the program exits.



实际上，为了得到上面展示的汇编代码，要使用`-fno-stack-protector`选项编译源码。这次不用这个选项重新编译。
    
    _echo:
    00000000000000b0    pushq   %rbp
    00000000000000b1    movq    %rsp,%rbp
    00000000000000b4    subq    $0x20,%rsp
    00000000000000b8    movq    ___stack_chk_guard(%rip),%rax //计算canary
    00000000000000bf    movq    (%rax),%rax
    00000000000000c2    movq    %rax,0xf8(%rbp)               //存到栈上，%rbp-8
    00000000000000c6    leaq    0xf0(%rbp),%rax
    00000000000000ca    movq    %rax,%rcx
    00000000000000cd    movq    %rcx,%rdi
    00000000000000d0    movq    %rax,0xe8(%rbp)
    00000000000000d4    callq   __gets
    00000000000000d9    movq    0xe8(%rbp),%rax
    00000000000000dd    movq    %rax,%rdi
    00000000000000e0    callq   _puts
    00000000000000e5    movq    ___stack_chk_guard(%rip),%rax //再次计算canary
    00000000000000ec    movq    (%rax),%rax
    00000000000000ef    movq    0xf8(%rbp),%rcx               //取出栈上的canary
    00000000000000f3    cmpq    %rcx,%rax                     //检查是否有变化
    00000000000000f6    jne     0x000000fe
    00000000000000f8    addq    $0x20,%rsp
    00000000000000fc    popq    %rbp
    00000000000000fd    ret
    0000000000000fe     callq   ___stack_chk_fail              //栈破坏！

新的栈结构图

![](http://farm9.staticflickr.com/8354/8363553569_f3bba35f1e.jpg)

这个新的[canary](http://en.wikipedia.org/wiki/Atlantic_Canary#Relationship_with_humans)，就是哨兵，由程序运行时随机产生。在从函数返回之前，程序检测这个值以确认栈未被破坏，达到避免攻击的目的。
