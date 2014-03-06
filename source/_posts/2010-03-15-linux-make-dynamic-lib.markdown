---
author: admin
comments: true
date: 2010-03-15 09:29:12+00:00
layout: post
published: false
slug: linux-make-dynamic-lib
title: linux编译生成动态库
wordpress_id: 251
categories:
- 默认
tags:
- 动态库
---

工作交接，手头没事闲的慌，试了一下怎么在Linux下生成动态库。放狗一搜，[铺天盖地的文章](http://www.google.cn/search?hl=zh-CN&newwindow=1&q=linux+动态库+创建&aq=f&aqi=g-g5g1g-g4&aql=&oq=)讲这个(PS：这年头，好像所有的东西都有人介绍过)。随便点了篇照着试了一下，很容易就搞定。只是有一件事不太爽，最后一步测试的时候，我发现找不到生成的那个动态库，然后又觉得这个场景似曾相识。在记忆里google了一下，发现原来N天之前我已经试验过了，结果现在又全还给google。为避免下次继续出现这种情况，还是把自己的实验过程写出来，免得再还回去。

实验开始，先准备一个要封成动态库的类。

头文件：

[cpp]
//myclass.h
class CMyClass
{
public:
   int add(int x, int y);
};
[/cpp]

实现：

[cpp]
//myclass.cpp
#include "myclass.h"

int CMyClass::add(int x, int y)
{
   return x + y;
}
[/cpp]


把文件编译成.so动态库：  **g++ myclass.cpp -fPIC -shared -o libmycls.so**




OK，动态库有了，来看看怎么用。




[cpp]
//main.cpp

#include "myclass.h"
#include <iostream>
using namespace std;

int main(void)
{
    CMyClass mycls;
    cout<<"10 plus 3 is "<<mycls.add(10,3)<<endl;
}
[/cpp]






编译main.cpp： **g++ main.cpp -L. -lmycls -o main**




看看生成的main能不能链接到我们的动态库：ldd main




如果像这样







linux-gate.so.1 =>  (0x005e5000)




libmycls.so => /home/user1/sourcecode/dlltest/libmycls.so (0x00ad8000)




libstdc++.so.6 => /usr/lib/libstdc++.so.6 (0x0039d000)




libm.so.6 => /lib/tls/i686/cmov/libm.so.6 (0x00951000)




libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x00cdc000)




libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0x00698000)




/lib/ld-linux.so.2 (0x00e12000)




第二行列出了之前创建的动态库，连接正常，执行main，看看结果吧。







附上-shared和-fPIC两个GCC选项的作用：







**-fPIC：如果支持这种目标机,编译器就输出位置无关目标码.适用于动态连接(dynamic linking),即使分支需要大范围转移.**







**-shared：生成一个共享目标文件,他可以和其他目标文件连接产生可执行文件.只有部分系统支持该选项.**










其实我在列出main的依赖项的时候碰到问题了，ldd main怎么也没列出来libmycls.so，后来设置环境变量才解决。打开~/.bashrc，增加两行($HOME是你的用户目录，$HOME/sourcecode/dlltest是动态库所在的目录)：







LD_LIBRARY_PATH=$HOME/sourcecode/dlltest




export LD_LIBRARY_PATH




保存退出，在命令行里面输入：source ~/.bashrc，让刚添加的环境变量生效。再来一次ldd main吧，现在应该没问题了。



