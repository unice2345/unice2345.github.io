---
layout: post
title:  "DCP57-CPP 不要从析构函数或内存解分配函数中抛出异常"
date:   2019-03-18 06:26:54 +0800
categories: jekyll update
---

在特定的条件下，通过抛出异常结束析构函数，operator delete,或operator delete[]会触发[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).


比如，C++标准,[basic.stc.dynamic.deallocation],第3段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]部分阐述如下：
> 如果一个析构函数通过抛出异常来结束，其行为是未定义的。

在这些场景下，函数在逻辑上必须声明为noexcept，因为从这些函数中抛出异常永远不会用正常定义的行为。C++标准，[except.spec]，第15段，阐述如下：
> 一个没有显示异常规定的析构函数，被认为是由noexcept(true)指定。

如上，内存解分配函数（对象，数组，和全局作用域或类作用域的占位形式的）不能通过抛出异常来终止。
不要将这些函数声明为noexcept(false)。
然而，依赖隐式的noexcept(true)规定或在函数签名中显示声明为noexcept是可以接受的。

异常抛出时会进行栈展开，栈展开过程中对象的析构函数会被调用。
如果在异常抛出时引起的析构函数调用中，又一次抛出了异常，那么函数std::terminate()会被调用，
函数std::terminate()默认情况下会调用std::abort() [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]。
当std::abort()被调用时，对象不再被销毁，导致程序处于未决定的状态和未定义的行为。
不要通过抛出异常结束一个析构函数。

C++标准，[class.dtor]，第3段，说明[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]如下：
> 一个没有异常规定的析构函数的声明，被隐式的认为和一个隐式声明具有同样的异常规定。

根据[except.spec]，第14段，一个析构函数的声明被认为隐式的声明为noexcept(true)。
所以，析构函数不能被声明为noexcept(false)，但是可以依赖隐式的noexcept(true)或显示声明为noexcept。

任何noexcept函数通过抛出异常来结束都会违反[ERR55-CPP. Honor exception specifications.](https://wiki.sei.cmu.edu/confluence/display/cplusplus/ERR55-CPP.+Honor+exception+specifications).

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，类的析构函数不满足隐式的noexcept规范，因为它可能抛出异常，即使是作为一个抛出异常的结果被调用。结果是，它被声明为noexcept(false),但是仍然可以触发[未定义行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).

{% highlight c++ %}

#include <stdexcept>
  
class S {
  bool has_error() const;
  
public:
  ~S() noexcept(false) {
    // Normal processing
    if (has_error()) {
      throw std::logic_error("Something bad");
    }
  }
};
{% endhighlight %}


# 不遵从规范的代码示例(std::uncaught_exception())

在析构函数中使用std::uncaught_exception()，通过在已经存在的被处理的异常时避免异常的传播解决了程序终止问题，如示例代码中所展示的。
但是，通过规避正常的析构函数处理流程，这个方法可能会使析构函数错过释放重要的资源。

{% highlight c++ %}
#include <exception>
#include <stdexcept>
  
class S {
  bool has_error() const;
  
public:
  ~S() noexcept(false) {
    // Normal processing
    if (has_error() && !std::uncaught_exception()) {
      throw std::logic_error("Something bad");
    }
  }
};
{% endhighlight %}


# 不遵从规范的代码示例(函数try块)

在这个不遵从规范的代码示例中，以及后面的遵从规范的解决方案中，假定存在一个Bad类，它的析构函数可以抛出异常。尽管这个类违反了本条规则，但是假定这个类不能被修改。

{% highlight c++ %}

// Assume that this class is provided by a 3rd party and it is not something
// that can be modified by the user.
class Bad {
  ~Bad() noexcept(false);
};
{% endhighlight %}


为了安全地使用Bad类，SomeClass的析构函数试图吸收掉Bad析构函数抛出的异常。

{% highlight c++ %}
class SomeClass {
  Bad bad_member;
public:
  ~SomeClass()
  try {
    // ...
  } catch(...) {
    // Handle the exception thrown from the Bad destructor.
  }
};
{% endhighlight %}


然而，C++标准，[except.handle], 第15段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]，
部分阐述如下：
> 如果控制到达构造函数或析构函数的函数try块的处理器的结尾时，当前处理的异常会重新抛出。

结果是，捕捉到的异常会不不可避免地再从SomeClass的析构函数逃掉，因为到达函数try块处理器末尾的时候异常会被隐式地重新抛出。


# 遵从规范的解决方案

一个析构函数在是否存在活动的异常时都应该执行同样的操作行为。
通常地，这意味着析构函数只应该调用哪些不会抛出异常的操作，或者它会处理掉所有的异常而不重新抛出它们(即使在隐式的情况下)。
这个遵从规范的解决方案不同于前面的不遵从规范的代码示例，它在SomeClass的析构函数中有一个显示的return语句。这个语句阻止了控制到达异常处理器的末尾。
结果是，当bad_member被销毁时，这个处理器可以捕获Bad::~Bad()抛出的异常。
它也会捕获函数try块中抛出的任何异常，但是SomeClass析构函数不会通过抛出异常而结束。

{% highlight c++ %}
class SomeClass {
  Bad bad_member;
public:
  ~SomeClass()
  try {
    // ...
  } catch(...) {
    // Catch exceptions thrown from noncompliant destructors of
    // member objects or base class subobjects.
 
    // NOTE: Flowing off the end of a destructor function-try-block causes
    // the caught exception to be implicitly rethrown, but an explicit
    // return statement will prevent that from happening.
    return;
  }
};
{% endhighlight %}

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，一个全局的解分配函数被声明为noexcept(false)并在某些条件不满足时抛出了异常。然而，从解分配函数中抛出异常会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)

{% highlight c++ %}
#include <stdexcept>
  
bool perform_dealloc(void *);
  
void operator delete(void *ptr) noexcept(false) {
  if (perform_dealloc(ptr)) {
    throw std::logic_error("Something bad");
  }
}
{% endhighlight %}


# 遵从规范的解决方案

这个遵从规范的解决方案中，在解分配操作发生失败时并没有抛出异常，而是尽可能优雅地结束掉。

{% highlight c++ %}

#include <cstdlib>
#include <stdexcept>
  
bool perform_dealloc(void *);
void log_failure(const char *);
  
void operator delete(void *ptr) noexcept(true) {
  if (perform_dealloc(ptr)) {
    log_failure("Deallocation of pointer failed");
    std::exit(1); // Fail, but still call destructors
  }
}
{% endhighlight %}


# 风险评估

从析构函数或解分配函数中抛出异常会导致未定义的行为，引起资源泄露或[拒绝服务](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-denial-of-service)攻击。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL57-CPP|低|很可能|中|P6|L2|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682) |[ERR55-CPP. Honor exception specifications](https://wiki.sei.cmu.edu/confluence/display/cplusplus/ERR55-CPP.+Honor+exception+specifications)|
|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682) |[ERR50-CPP. Do not abruptly terminate the program](https://wiki.sei.cmu.edu/confluence/display/cplusplus/ERR50-CPP.+Do+not+abruptly+terminate+the+program)|
|[MISRA C++:2008](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-MISRA08)|Rule 15-5-1 (Required)|

# 参考书目

|[Henricson 1997](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Henricson97)|Recommendation 12.5, Do not let destructors called during stack unwinding throw exceptions|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 3.4.7.2, "Deallocation Functions"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 15.2, "Constructors and Destructors"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 15.3, "Handling an Exception" |
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 15.4, "Exception Specifications" |
|[Meyers 2005](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Meyers05)|Item 8, "Prevent Exceptions from Leaving Destructors"|
|[Sutter 2000](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Sutter00)|"Never allow exceptions from escaping destructors or from an overloaded operator delete()" (p. 29)
|

# 参考链接

[DCL57-CPP. Do not let exceptions escape from destructors or deallocation functions][1]

[noexcept_spec][2]

[Exceptions and Stack Unwinding in C++][3]

[what-is-stack-unwinding][4]

[Stack unwinding (C++ only)][5]

[std::terminate][6]

[Stack unwinding][7]

[Function-try-block][8]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL57-CPP.+Do+not+let+exceptions+escape+from+destructors+or+deallocation+functions

[2]: https://en.cppreference.com/w/cpp/language/noexcept_spec

[3]: https://docs.microsoft.com/en-us/cpp/cpp/exceptions-and-stack-unwinding-in-cpp?view=vs-2017

[4]: https://stackoverflow.com/questions/2331316/what-is-stack-unwinding

[5]: https://www.ibm.com/support/knowledgecenter/SSLTBW_2.1.0/com.ibm.zos.v2r1.cbclx01/cplr155.htm

[6]: https://en.cppreference.com/w/cpp/error/terminate

[7]: https://en.cppreference.com/w/cpp/language/throw#Stack_unwinding

[8]: https://en.cppreference.com/w/cpp/language/function-try-block