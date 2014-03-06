---
author: admin
comments: true
date: 2010-03-11 08:19:47+00:00
layout: post
slug: helpmacros-in-cppunit
title: cppunit中的帮助宏
published: false
wordpress_id: 230
categories:
- 默认
tags:
- cppunit
- 单元测试
---




本文来自include\cppunit\extensions\HelperMacros.h头文件，这里只是做简单的翻译，少数地方加入一些自己的理解。




从[cppunit cookbook](http://www.huubby6.tk/2010/01/27/cppunit-cookbook-chinese/)里，我们了解了cppunit的使用方法，其中牵涉到cppunit预定义的几个宏，CPPUNIT_TEST_SUITE()，CPPUNIT_TEST()，CPPUNIT_TEST_SUITE_END()和CPPUNIT_TEST_SUITE_NAMED_REGISTRATION()。这些宏由cppunit提供，以帮助使用者更加方便的定义一个test suite。来具体看看这几个宏的作用。


[cpp]
class CTDemoMfc_CDemo : public CppUnit::TestCase{
            CPPUNIT_TEST_SUITE(CTDemoMfc_CDemo);
            CPPUNIT_TEST(TestAdd);
            CPPUNIT_TEST(TestSubtration);
            CPPUNIT_TEST_SUITE_END();
public:
	CTDemoMfc_CDemo(void);
	void TestAdd();
	void TestSubtration();
	static std::string RegCaseName() { return "CTDemoMfc_CDemo";}
public:
	~CTDemoMfc_CDemo(void);
};
[/cpp]






这段代码里面，CPPUNIT_TEST_SUITE()和CPPUNIT_TEST_SUITE_END()两个宏的作用就是在类CTDemoMfc_CDemo中添加两个方法。一个辅助性的addTestsToSuite方法，添加测试项到test suite里面，另一个static CppUnit::TestSuite *suite()方法，返回一个指向由CPPUNIT_TEST定义的一系列test的指针。




在类CTDemoMfc_CDemo的实现文件里面，还有一个宏：CPPUNIT_TEST_SUITE_NAMED_REGISTRATION()，它创建了一个AutoRegisterSuite类的静态对象，自动将test suite注册到一个全局的registry中。这个全局的registry可以生成一个Test实例，包含了所有的已注册的test suite。




[cpp]
CPPUNIT_TEST_SUITE_NAMED_REGISTRATION(CTDemoMfc_CDemo,CTDemoMfc_CDemo::RegCaseName());
CppUnit::MfcUi::TestRunner runner;
CppUnit::TestFactoryRegistry &registry =
                 CppUnit::TestFactoryRegistry::getRegistry(CTDemoMfc_CDemo::RegCaseName());
runner.addTest(registry.makeTest());
runner.run();
[/cpp]






test suite宏也可用于模板测试类




[cpp]
template<typename CharType>
class StringTest : public CppUnit::TestCase {
   CPPUNIT_TEST_SUITE( StringTest );
   CPPUNIT_TEST( testAppend );
   CPPUNIT_TEST_SUITE_END();
public:
   ...
};
[/cpp]






在实现文件中添加代码：




[cpp]
CPPUNIT_TEST_SUITE_NAMED_REGISTRATION( StringTest<char> );
CPPUNIT_TEST_SUITE_NAMED_REGISTRATION( StringTest<wchar_t> );
[/cpp]

<!-- more -->






除了这几个宏外，cppunit中还有很多其他的宏，逐个来看。




_CPPUNIT_TEST_SUITE(ATestFixtureType)_：开始声明一个新的test suite。当你想包括父类的test suite的时候，请使用CPPUNIT_TEST_SUB_SUITE()宏代替。参数ATestFixtureType表示测试类，注意，测试类必须是TestFixture的子类；
**_CPPUNIT_TEST_SUB_SUITE(ATestFixtureType, ASuperClass)_**：开始一个新的test suite，包括父类的test suite。只有当前测试类的父类使用CPPUNIT_TEST_SUITE()或CPPUNIT_TEST_SUB_SUITE()定义test suite时，才能使用这个宏。这个宏按照CPPUNIT_TEST_SUITE()一样的方式开始声明新的test suite，随后，父类的test suite被自动加入到当前的suite中。参数ATestFixtureType与CPPUNIT_TEST_SUITE中的参数意义一样，参数ASuperClass表示父类。
**_CPPUNIT_TEST_SUITE_END()_**:结束test suite声明。注意的是，在这个宏之后的成员，若未重新指定访问权限，将设置为private。出于测试代码易读的考虑，建议在这个宏后面加上明确的访问权限，不论紧跟其后的成员是public,protected还是private。
**_CPPUNIT_TEST_SUITE_END_ABSTRACT()_**：结束纯虚test suite的声明。用这个宏表明此TestFixture是纯虚的，未声明静态的suite()方法。与CPPUNIT_TEST_SUITE_END()一样，在这个宏之后的成员，若未重新指定访问权限，同样将设置为private。用一个例子来说明这个宏的用法：




[cpp]
//The abstract test fixture
#include <cppunit/extensions/HelperMacros.h>
class AbstractDocument;
class AbstractDocumentTest : public CppUnit::TestFixture {
  CPPUNIT_TEST_SUITE( AbstractDocumentTest );
  CPPUNIT_TEST( testInsertText );
  CPPUNIT_TEST_SUITE_END_ABSTRACT();
public:
  void testInsertText();
  void setUp()
  {
    m_document = makeDocument();
  }
  void tearDown()
  {
    delete m_document;
  }
protected:
  virtual AbstractDocument *makeDocument() =0;
  AbstractDocument *m_document;
};

//The concrete test fixture
class RichTextDocumentTest : public AbstractDocumentTest {
   CPPUNIT_TEST_SUB_SUITE( RichTextDocumentTest, AbstractDocumentTest );
   CPPUNIT_TEST( testInsertFormatedText );
   CPPUNIT_TEST_SUITE_END();
public:
   void testInsertFormatedText();
protected:
   AbstractDocument *makeDocument()
   {
     return new RichTextDocument();
   }
};
[/cpp]






_CPPUNIT_TEST_SUITE_ADD_TEST(test)_：添加一个test到suite中(用于自定义新的宏)，参数test将被添加到声明的suite中。这个宏设计用来定制新的更高级的宏，以方便对cppunit进行扩展。在CPPUNIT_TEST_SUITE()和CPPUNIT_TEST_SUITE_END()两个宏之间，可以使用两个变量




[cpp]
typedef TestSuiteBuilder<TestFixtureType> TestSuiteBuilderType;
TestSuiteBuilderType &context;
[/cpp]






context可以用来命名test case，创建新的test fixture实例，添加test case到test suite。这个宏就是用context将test添加到suite中。




用一个例子来说明如何使用这个宏创建新的宏，将test添加到suite中。在这里例子里面，订制的新宏将一个test添加到suite中，当这个test执行超过给定的时间后，即认为失败。这个宏基于一个与TestCaller有同样接口的虚构的TimeOutTestCaller类。




[cpp]
#define CPPUNITEX_TEST_TIMELIMIT( testMethod, timeLimit )      \
     CPPUNIT_TEST_SUITE_ADD_TEST( (new TimeOutTestCaller<TestFixtureType>( \
                  namer.getTestNameFor( #testMethod ),         \
                  &TestFixtureType::testMethod,                \
                  factory.makeFixture(),                       \
                  timeLimit ) ) )

class PerformanceTest : CppUnit::TestFixture
{
       CPPUNIT_TEST_SUITE( PerformanceTest );
       CPPUNITEX_TEST_TIMELIMIT( testSortReverseOrder, 5.0 );
       CPPUNIT_TEST_SUITE_END();
public:
       void testSortReverseOrder();
};
[/cpp]






_CPPUNIT_TEST(testMethod)_：添加一个方法testMethod到test suite中。testMethod的原型必须为void testMethod();




**_CPPUNIT_TEST_EXCEPTION( testMethod, ExceptionType )_**：添加一个test到test suite中。当指定的异常未被捕获时，这个test会失败。同样，再通过一个例子来看看这个宏的用法。




[cpp]
#include <cppunit/extensions/HelperMacros.h>
#include <vector>
class MyTest : public CppUnit::TestFixture {
   CPPUNIT_TEST_SUITE( MyTest );
   CPPUNIT_TEST_EXCEPTION( testVectorAtThrow, std::invalid_argument );
   CPPUNIT_TEST_SUITE_END();
public:
   void testVectorAtThrow()
   {
     std::vector<int> v;
     v.at(1);     // must throw exception std::invalid_argument
   }
};
[/cpp]






参数testMethod表示要添加到test suite当中的test的名称；参数ExceptionType即为testMethod要抛出的异常类型。这个宏在cppunit当中不建议使用，可以使用CPPUNIT_ASSERT_THROW宏代替。




_CPPUNIT_TEST_SUITE_REGISTRATION( ATestFixtureType )_：添加指定的fixture到未命名的registry中。这个宏声明了一个静态对象，其构造函数将一个test suite工厂加入到这类工厂的全局registry当中。可以通过调用函数CppUnit::TestFactoryRegistry::getRegistry()获取这个全局的registry。参数ATestFixtureType表示测试类。要注意的是，这个宏在一行内只能用一次，因为行号将会被用来命名一个隐藏的静态变量。




**_CPPUNIT_TEST_SUITE_NAMED_REGISTRATION( ATestFixtureType, suiteName )_**：添加指定的fixture到指定的registry中。这个宏声明了一个静态对象，其构造函数将一个test suite工厂加入到由参数suiteName指定名字的全局registry当中。可以通过调用函数CppUnit::TestFactoryRegistry::getRegistry()获取这个全局的registry。建议使用静态的方法返回字符串作为suiteName参数的值，这样比直接硬编码字符串要好，可以避免输入上的错误。




[cpp]
// MySuites.h
namespace MySuites {
  std::string math() {
    return "Math";
  }
}

// ComplexNumberTest.cpp
#include "MySuites.h"

CPPUNIT_TEST_SUITE_NAMED_REGISTRATION( ComplexNumberTest, MySuites::math() );
[/cpp]






参数ATestFixtureType表示测试类。要注意的是，这个宏在一行内只能用一次，因为行号将会被用来命名一个隐藏的静态变量。




经常使用的帮助宏就介绍到这里，还有其他的一些宏，可以参照HelperMacros.h头文件中的说明。文中所涉及的宏，我并未全部使用，若翻译中有错误，欢迎指正。
