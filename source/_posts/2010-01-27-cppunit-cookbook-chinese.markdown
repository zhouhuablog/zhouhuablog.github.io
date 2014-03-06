---
author: admin
comments: true
date: 2010-01-27 14:33:01+00:00
layout: post
slug: cppunit-cookbook-chinese
published: false
title: CppUnit CookBook中文版
wordpress_id: 197
categories:
- 默认
tags:
- cookbook
- cppunit
- 单元测试
---

自己翻译的cppunit cookbook，如有错漏，欢迎指出。可以在[这里](http://sourceforge.net/projects/cppunit/)下载到cppunit的最新版本源码。

这只是一个cookbook的翻译，并没介绍安装方法，你可以在[这里](http://www.vckbase.com/document/viewdoc/?id=1258)找到win32下的安装方法和例子。不过，这个例子并不清楚，还是建议你看看[这里](http://blog.csdn.net/jinjunmax_78/archive/2008/01/10/2033755.aspx)的例子，清楚的多。看完这些安装方法和例子之后，再回头看看这篇cookbook，应该会帮助你理解例子里面那些代码的含义。

1、简单的测试案例

怎么才能知道你的代码是不是能够正常工作？有很多方法可以达到这个目的。通过调试器单步跟踪，或者在你的代码里加入打印输出代码是两个比较简单的办法，但是这两个方法都有缺点。单步跟踪不能自动进行，每次代码稍有调整就要进行调试。打印输出也不错，只是这种方法会增加很多不必要的代码，导致代码臃肿丑陋。

CppUnit单元测试很容易建立起来，并且可以自动进行，而且，一旦你写完测试用例，就能通过它们保证你的代码质量。

按照下面的流程可以构造一个简单的test：

1、继承CppUnit::TestCase类。

2、重写runTest()方法。

3、使用CPPUNIT_ASSERT()和CPPUNIT_ASSERT(bool)两个宏来检测表达式或值，以判断测试成功与否。

举个例子，如果要测试一个复数类的赋值(=号)运算符，按照上面的步骤，代码如下：

[cpp]class ComplexNumberTest : public CppUnit::TestCase
{
public:
     ComplexNumberTest( std::string name ) : CppUnit::TestCase( name ) {}
     void runTest()
     {
          CPPUNIT_ASSERT( Complex (10, 1) == Complex (10, 1) );
          CPPUNIT_ASSERT( !(Complex (1, 1) == Complex (2, 2)) );
     }
};[/cpp]

一个简单的test就建起来了。但通常来说，我们会在同一个对象里面有很多小的测试用例。这种情况下，我们用fixture。

2、fixture

fixture是为一组测试用例提供基础服务的对象，当你边开发边测试时，使用fixture非常方便。我们试着模拟一下这种边开发边测试情况。

假设我们真的在开发一个复数类，首先，定义一个空的Complex类：

[cpp]class Complex {};[/cpp]

现在，创建一个上面的ComplexNumberTest类对象，然后编译代码看看会发生什么。我们会得到几个编译错误。因为测试过程中用到了==操作符，但我们并没定义这个运算符。

现在为Complex类定义一个：

[cpp]bool operator==( const Complex &amp;a, const Complex &amp;b)
{
     return true;
}[/cpp]

再次编译运行这个测试。这次编译虽然通过了，但测试却是失败的。

要再做一点点事情让==操作符正常工作，重写代码如下：

[cpp]class Complex
{
     friend bool operator ==(const Complex&amp; a, const Complex&amp; b);
     double real, imaginary;
public:
     Complex( double r, double i = 0 ) :real(r) , imaginary(i) {}
};
bool operator ==( const Complex &amp;a, const Complex &amp;b )
{
     return a.real == b.real  &amp;&amp;  a.imaginary == b.imaginary;
}
[/cpp]

编译运行，测试顺利通过。

好了，现在我们准备增加一些新的操作和新的测试。这个时候，fixture就体现出方便性了。因为在这个时候，如果使用fixture实例化三四个Complex对象，并且在测试过程中复用这几个对象，这样不需要重复构建这些实例对象，测试代码会更加好写。

fixture的使用步骤如下：

- 给每个fixture添加成员变量；

- 重写CppUnit::TestFixture::setUp()以初始化这些变量；

- 重写CppUnit::TestFixture::tearDown()方法释放setUp申请的系统资源。

[cpp]class ComplexNumberTest : public CppUnit::TestFixture
{
private:
     Complex *m_10_1, *m_1_1, *m_11_2;
public:
    void setUp()
    {
         m_10_1 = new Complex( 10, 1 );
         m_1_1 = new Complex( 1, 1 );
         m_11_2 = new Complex( 11, 2 );
    }
    void tearDown()
    {
         delete m_10_1;
         delete m_1_1;
         delete m_11_2;
     }
};[/cpp]

有了这个fixture之后，我们就能在开发过程中添加另外的测试用例和我们需要的其他东西了。

<!-- more -->

3、Test Case

怎么用fixture编写和调用单独的测试呢？

可以分两步完成：

- 把要调用的测试写成fixture类的一个方法；

- 创建一个运行指定方法的TestCaller

下面是一个有了测试方法的test case：

[cpp]class ComplexNumberTest : public CppUnit::TestFixture
{
private:
     Complex *m_10_1, *m_1_1, *m_11_2;
public:
     void setUp()
     {
          m_10_1 = new Complex( 10, 1 );
          m_1_1 = new Complex( 1, 1 );
          m_11_2 = new Complex( 11, 2 );
     }
     void tearDown()
     {
          delete m_10_1;
          delete m_1_1;
          delete m_11_2;
     }
    void testEquality()
    {
          CPPUNIT_ASSERT( *m_10_1 == *m_10_1 );
          CPPUNIT_ASSERT( !(*m_10_1 == *m_11_2) );
    }
    void testAddition()
    {
         CPPUNIT_ASSERT( *m_10_1 + *m_1_1 == *m_11_2 );
    }
};[/cpp]

可以像下面这样创建并调用每个test case:

[cpp]CppUnit::TestCaller&lt;ComplexNumberTest&gt; test(&quot;testEquality&quot;, &amp;ComplexNumberTest::testEquality );
CppUnit::TestResult result;
test.run( &amp;result );[/cpp]

TestCaller构造函数的第二个参数是ComplexNumberTest类中某个方法的地址。当TestCaller运行的时候，指定的方法也就运行了。

但是，这些都是没用的，因为没有显示判断结果。通常，我们使用TestRunner来显示测试结果。(后面会讲到)

当你有很多测试用例的时候，可以把它们组织成test suite。

4、Suite

我们在上面已经知道如何运行单个的test case，那怎么才能让所有的Test Case一次性全部运行起来呢？

CppUnit提供了一个TestSuite，可以把很多Test Case组织在一起运行。

可以按照下面的方式把两三个test组织成一个test suite：

[cpp]CppUnit::TestSuite suite;
CppUnit::TestResult result;
suite.addTest( new CppUnit::TestCaller&lt;ComplexNumberTest&gt;(&quot;testEquality&quot;,&amp;ComplexNumberTest::testEquality ) );
suite.addTest( new CppUnit::TestCaller&lt;ComplexNumberTest&gt;(&quot;testAddition&quot;, &amp;ComplexNumberTest::testAddition ) );
suite.run( &amp;result );[/cpp]

TestSuites不仅仅可以包含TestCases的TestCaller，它还能包含任何实现了CppUnit::Test接口的对象。例如，你可以在你的代码里面创建一个TestSuite，我也可以在我的代码里面创建一个TestSuite，我们可以创建一个包含这两个TestSuite的新的TestSuite，然后你我两个TestSuite就可以一起执行了。

[cpp]CppUnit::TestSuite suite;
CppUnit::TestResult result;
suite.addTest( ComplexNumberTest::suite() );
suite.addTest( SurrealNumberTest::suite() );
suite.run( &amp;result );[/cpp]

5、TestRunner

怎么运行测试用例并收集结果？

有了一个TestSuite，你肯定想去运行它。CppUnit提供了运行suite并显示结果的工具-TestRunner。在你的suite中添加一个静态方法，获取test suite，这样你的suite就能够被TestRunner调用了。

例如，要使TestRunner能够调用ComplexNumberTest 这个test suite，可以在ComplexNumberTest中添加如下代码：

[cpp]public:
static CppUnit::TestSuite *suite()
{
     CppUnit::TestSuite *suiteOfTests = new CppUnit::TestSuite( &quot;ComplexNumberTest&quot; );
     suiteOfTests-&gt;addTest( new CppUnit::TestCaller&lt;ComplexNumberTest&gt;(&quot;testEquality&quot;, &amp;ComplexNumberTest::testEquality ) );
     suiteOfTests-&gt;addTest( new CppUnit::TestCaller&lt;ComplexNumberTest&gt;(&quot;testAddition&quot;, &amp;ComplexNumberTest::testAddition ) );
     return suiteOfTests;
}[/cpp]

下面是使用test runner调用test suite的代码，以文本界面的test runner为例。

要使用文本界面的TestRunner，请在main.cpp中包含头文件：

[cpp]#include &lt;cppunit/ui/text/TestRunner.h&gt;
#include &quot;ExampleTestCase.h&quot;
#include &quot;ComplexNumberTest.h&quot;[/cpp]

并且在main函数中添加调用addTest方法的代码：

[cpp]int main( int argc, char **argv)
{
    CppUnit::TextUi::TestRunner runner;
    runner.addTest( ExampleTestCase::suite() );
    runner.addTest( ComplexNumberTest::suite() );
    runner.run();
    return 0;
}[/cpp]

这样，TestRunner就会运行所有的测试了。如果过程中有任何一个测试失败，会得到下面的信息：

- 失败测试的名字；

- 失败测试源文件名；

- 失败测试失败的行数；

- 检测到测试失败的CPPUNIT_ASSERT宏内部的所有文本信息。

CppUnit区分两种错误，failure和error。failure是框架预期的结果，会被断言宏检测到。而error则是非预期的，比如除0，或者其他C++运行时抛出的异常，你自己的代码抛出的异常等，这些不是框架预期的错误。

6、帮助宏

你可能已经注意到了，实现fixture的静态suite()方法是个重复且容易出错的工作。CppUnit创建了一系列WritingTestFixture宏可以帮助你自动实现静态的suite()方法。

下面是用这些宏重写之后的ComplexNumberTest类代码：

[cpp]#include &lt;cppunit/extensions/HelperMacros.h&gt;

class ComplexNumberTest : public CppUnit::TestFixture
{
    //首先，我们声明suite，把类名传给宏
    CPPUNIT_TEST_SUITE( ComplexNumberTest );
    //然后，我们声明fixture的每个test case，即添加测试函数到suite中
    CPPUNIT_TEST( testEquality );
    CPPUNIT_TEST( testAddition );
    //最后，结束suite声明
    CPPUNIT_TEST_SUITE_END();
    //这个时候，static CppUnit::TestSuite *suite();这个签名的方法已经实现了

    //剩下的代码原样保留
private:
    Complex *m_10_1, *m_1_1, *m_11_2;
public:
    void setUp()
    {
         m_10_1 = new Complex( 10, 1 );
         m_1_1 = new Complex( 1, 1 );
         m_11_2 = new Complex( 11, 2 );
    }

    void tearDown()
    {
         delete m_10_1;
         delete m_1_1;
         delete m_11_2;
     }

     void testEquality()
     {
          CPPUNIT_ASSERT( *m_10_1 == *m_10_1 );
          CPPUNIT_ASSERT( !(*m_10_1 == *m_11_2) );
     }
     void testAddition()
     {
          CPPUNIT_ASSERT( *m_10_1 + *m_1_1 == *m_11_2 );
     }
};[/cpp]

添加到suite中的TestCaller名称是由fixture名称和方法名符合而成的。

例如，在当前的test case中，TestCaller的名称是："ComplexNumberTest.testEquality"和"ComplexNumberTest.testAddition"。

帮助宏能够帮你编写普通的断言。例如，检查ComplexNumber中发生除0异常时，是否抛出MathException异常：

- 使用CPPUNIT_TEST_EXCEPTION宏，指定预期的异常类型，编写测试并添加到suite中

- 在fixture中编写测试方法

[cpp]CPPUNIT_TEST_SUITE( ComplexNumberTest );
// [...]
CPPUNIT_TEST_EXCEPTION( testDivideByZeroThrows, MathException );
CPPUNIT_TEST_SUITE_END();
// [...]
void testDivideByZeroThrows()
{
     // 下面的代码会抛出MathException.
     *m_10_1 / ComplexNumber(0);
}[/cpp]

若没抛出预期的异常，则会报告一个assert failure。

7、TestFactoryRegistry

TestFactoryRegistry解决了两个麻烦：

- 忘记向test runner中添加fixture（由于两者不在同一个文件中，确实很容易忘记）

- 包含所有测试头文件的麻烦

TestFactoryRegistry是程序初始化时注册test suites的地方。

在cpp文件中添加下面的代码，注册ComplexNumber suite：

[cpp]#include &lt;cppunit/extensions/HelperMacros.h&gt;

CPPUNIT_TEST_SUITE_REGISTRATION( ComplexNumberTest );[/cpp]

在这句代码背后，声明了一个静态的CppUnit::AutoRegisterSuite变量。在构造函数中，它调用CppUnit::TestFactoryRegistry::registerFactory(TestFactory*)方法注册了一个CppUnit::TestSuiteFactory到CppUnit::TestFactoryRegistry中。TestSuiteFactory通过ComplexNumberTest::suite()(译者注：原文为ComplexNumber::suite()，疑有误，特改之)方法返回一个TestSuite。

使用文本模式的test runner运行测试，此时不再需要包含fixture：

[cpp]#include &lt;cppunit/extensions/TestFactoryRegistry.h&gt;
#include &lt;cppunit/ui/text/TestRunner.h&gt;

int main( int argc, char **argv)
{
    CppUnit::TextUi::TestRunner runner;
    //首先，查找的CppUnit::TestFactoryRegistry的实例
    CppUnit::TestFactoryRegistry &amp;registry = CppUnit::TestFactoryRegistry::getRegistry();
    //CppUnit::TestFactoryRegistry中包含所有使用
    //CPPUNIT_TEST_SUITE_REGISTRATION()宏注册的测试用例，
    //所以，可以通过TestFactoryRegistry获取一个CppUnit::TestSuite，并将其添加到test //runner中
    runner.addTest( registry.makeTest() );
    runner.run();
    return 0;
}[/cpp]

8、生成后测试

到现在为止，我们的单元测试可以正常运行了，可是怎样将单元测试整合到我们的构建过程中呢？

首先，应用程序必须返回一个不等于0的值，以表明有错误发生。

而TestRunner::run()方法返回一个布尔值，用来表示测试是否成功。

可以修改主程序如下：

[cpp]#include &lt;cppunit/extensions/TestFactoryRegistry.h&gt;
#include &lt;cppunit/ui/text/TestRunner.h&gt;

int main( int argc, char **argv)
{
    CppUnit::TextUi::TestRunner runner;
    CppUnit::TestFactoryRegistry &amp;registry = CppUnit::TestFactoryRegistry::getRegistry();
    runner.addTest( registry.makeTest() );
    bool wasSuccessful = runner.run( &quot;&quot;, false );
    return !wasSuccessful;
}[/cpp]

现在，编译后运行程序吧。

在VC++中，可以设置生成事件->生成后事件，指定命令行参数为“ "$(TargetPath)" ”达到将单元测试整合进构建过程中的目的。"$(TargetPath)"将会被展开为此测试工程的可执行文件路径，所以，当编译生成完成后，会自动运行main.exe完成测试。这样就完成了单元测试与构建过程的绑定。
