---
author: admin
comments: true
date: 2010-06-09 12:59:07+00:00
layout: post
slug: cpp-exception-handling
title: C++ Exception Handling
wordpress_id: 303
categories:
- 默认
tags:
- exception handling
---

最近刚刚读完《深度探索C++对象模型》，说来惭愧，整本书都快看完才想起来要做笔记。目前笔记只有后面三章的一些重点内容，本文是异常处理(Exception Handling)这部分内容的笔记整理。

写程序的时候，对于可能出现异常的地方，通常来说我们是这样写的：

    try{
    ...
    throw ...
    } catch(...) {
    //exception handling
    }

当异常发生时，代码跳转到catch块，完成异常处理后退出。为了支持这个处理流程，编译器要能够在发生异常时，找到对应的catch子句。而查找catch子句时，编译器要知道函数的当前作用范围，同时还要了解抛出异常的类型，以便进行catch子句匹配(这引出了执行期类型识别的需求，即RTTI)。

详细来看看，上面这段代码大致可以分为三个部分：

* `throw`子句，它抛出一个`exception`，可以是内置类型，也可以是用户自定义类型

* `catch`子句，每个`catch`子句都是一个`exception handler`，表示其之后的代码段用来处理某种类型的exception

* `try`段，内含一些程序代码，这些代码可能引发`catch`子句生效

当一个`exception`被抛出时，程序控制权从函数中释放，转而寻找一个匹配的`catch`子句。若不存在匹配的`catch`，默认的`terminate()`方法将被调用。而在控制权被放弃后，堆栈中的函数被poped up，函数内局部对象的析构函数在poped up之前会被调用。

当exception发生，编译系统完成下面几件必须要做的事：

1、检查发生`throw exception`操作的函数

2、决定`throw`是否是发生在一个try段内

3、如果throw发生在try段内，编译系统开始比较发生的exception的类型和catch子句中exception的类型是否吻合

4、一旦exception类型吻合，该catch子句就得到流程控制权

5、若throw并不是发生在try段内，或者没有找到吻合的catch子句，系统要做的就是，清理当前函数范围内所有局部对象，从堆栈中pop up当前函数，进入堆栈内下一个函数，继续步骤2-5

那么当一个exception被抛出后，它经历了些什么？它并没有无家可归，睡在天桥底下。抛出的exception object放在exception数据堆栈中，从throw处传递到catch子句的是这个exception object的地址。来看个例子：


    catch (BaseException  newobj) {
        //do something
        throw ;
    }


假设这个catch子句将收到一个`DerivedException`类型的exception object，`DerivedException`继承自`BaseException`。那么，由于exception类型吻合，按照上面说的步骤4，这个catch子句得到控制权。

catch子句中的newobj在收到exception时，将以抛出的exception object作为初值，像函数以传值方式传递参数一样。由于newobj是一个对象而不是指针或引用，BaseException的copy constructor将会被调用(如果存在的话)，以拷贝原始exception object中属于BaseException的部分到当前的newobj中。

完成这些之后，catch子句内的代码被执行，再次抛出异常。这次抛出的是newobj还是原始的exception object？抛出的仍然是原始的exception object。newobj只是局部对象，在catch子句结束后被摧毁，任何对newobj的修改都被丢弃而不会影响到原始的exception object。这个原始的exception object将直到有一个catch子句执行完，并且不再抛出exception后，才会被摧毁。

本文内容来自《深度探索C++对象模型》第7章中的异常处理介绍。

关于C++的EH，在codeproject上有篇[文章](http://www.codeproject.com/KB/cpp/Win32Exceptions.aspx)简单介绍编译系统与win32系统之间的协作关系。

that's all, over！
