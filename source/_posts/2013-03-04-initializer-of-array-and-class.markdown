---
author: admin
comments: true
date: 2013-03-04 08:41:47+00:00
layout: post
slug: initializer-of-array-and-class
title: 初始化:数组和类
wordpress_id: 510
categories:
- 默认
tags:
- array
- class
- initializer
---

前段时间有个项目，其中有一部分从磁盘读出数据的功能，我们用64块2M内存来保存读出的数据。测试的时候，发现读取速度非常之慢，与理想情况有1个数量级的差距。于是在查错调优的时候，发现了这个问题。



	
  * **怎么了？**


2M内存在代码中用`typedef`一个数组来表示：

    const int SIZE_OF_MEM = 2*1024*1024;
    typedef char Block[SIZE_OF_MEM];

简化读取逻辑，去掉其他细节之后，得到数据块使用部分的模拟代码：

    const int n = 100;
    void *p[n];
    for (int i = 0; i < n; ++i) { // 1
        p[i] = new Block();
    }
    for (int i = 0; i < 1000000 * n; ++i) { // 2
        p[i%n] = new(p[i%n]) Block(); //取余以减少测试需要的内存数量
    }
    for (int i = 0; i < n; ++i) {
        delete p[i];
    }
    return 0;
经历数次错误的怀疑/验证，通过对整体逻辑的一步步计时，最后发现中间的第二个`for`循环里的逻辑就是症结所在。平均每一次的`new-placement`耗时在毫秒级（milliseconds），大大超出预期，导致读取模型工作效率低下。

看起来数组`typedef`不灵光，猜测数组可能存在某些特性，导致`new-placement`运行效率远低于预期。换成类试试看:
    
    const int SIZE_OF_MEM=2*1024*1024;
    class Block
    {
    public:
    	Block() {}
    private:
    	char data[SIZE_OF_MEM];
    };

再一次计时，1,000,000次的执行时间不到200ms，看起来正常多了。

这张图对比了测试代码`new-placement`在`block`为array和class时的执行效率差异。其中，array由于耗时太久，只执行了1000次，而class执行1,000,000次。
![Comparison of Array and Class](http://farm9.staticflickr.com/8235/8526560021_284be2ce3b.jpg)

为什么定义成类比`typedef`的数组快这么多？看看`new-placement`语句的汇编代码怎么说（均为VS2008编译）。
![VS2008 assembly of array and class](http://farm9.staticflickr.com/8373/8526560091_79f3c12bdb_b.jpg)

汇编代码清楚的显示两种定义的不同。对于数组，在`new`操作完成后，有`memset`，而对于类，则有构造函数的调用。对于2M内存，数组类型在`new`完成后要被初始化为0，这就是低效率的罪魁祸首。

再来看看Linux下gcc的汇编代码：
![linux assembly of array and class](http://farm9.staticflickr.com/8109/8527725118_75cf398054_b.jpg)
GCC和VS2008编译结果一致，看起来应该不是编译器bug，:)



	
  * 为什么？


找找C++标准，关于初始化的部分有这么一条：


> An object whose initializer is an empty set of parentheses, i.e., (), shall be value-initialized.


什么是value-initialized？


> To value-initialize an object of type T means:

> 
> 
	
>   * if T is a (possibly cv-qualified) class type (Clause 9) with a user-provided constructor (12.1), then the default constructor for T is called (and the initialization is ill-formed if T has no accessible default constructor);
> 
	
>   * if T is a (possibly cv-qualified) non-union class type without a user-provided constructor, then the object is zero-initialized and, if T’s implicitly-declared default constructor is non-trivial, that constructor is called.
> 
>   * if T is an array type, then each element is value-initialized;
> 
	
>   * otherwise, the object is zero-initialized.
> 




这种空的小括号，()，一般情况下不能用来表示初始化某个对象。但是，在少数情况下，它是合法的，上面的`new`就属于这些少数情况里的一个。在这种情况下，要初始化的类型又是数组，于是不幸落入第三种情况，每个元素都被初始化为0。Linux下的汇编代码清楚的表明了这个过程：
    
    0x000000000044187c cmpq   $0x0,-0x18(%rbp)
    0x0000000000441881 je     0x4418b9
    0x0000000000441883 mov    -0x18(%rbp),%rax
    0x0000000000441887 mov    %rax,%rsi
    0x000000000044188a mov    $0x200000,%edi
    0x000000000044188f callq  0x43d853 
    0x0000000000441894 mov    %rax,%rdx
    0x0000000000441897 test   %rdx,%rdx
    0x000000000044189a je     0x4418b9
    0x000000000044189c mov    $0x1fffff,%edx               //数组大小
    0x00000000004418a1 jmp    0x4418ae
    0x00000000004418a3 movb   $0x0,(%rax)  	            // 置0
    0x00000000004418a6 add    $0x1,%rax                   //移动指针
    0x00000000004418aa sub    $0x1,%rdx                   //数组大小减1
    0x00000000004418ae cmp    $0xffffffffffffffff,%rdx    //判断是否完成全部元素置0
    0x00000000004418b2 setne  %cl
    0x00000000004418b5 test   %cl,%cl
    0x00000000004418b7 jne    0x4418a3
    0x00000000004418b9 mov    -0x18(%rbp),%rax
不同的初始化方式编译得到的汇编代码差异很大，如果在`new-placement`之后不写()，结果又如何？不同的优化等级会有影响么？有兴趣的可以查查标准，再自己动手试试。
