---
layout: post
title:  "DCL59-CPP 不要在头文件中定义匿名命名空间"
date:   2019-03-28 07:24:54 +0800
categories: jekyll update
---

匿名命名空间是用来定义一个翻译单元中唯一的一个命名空间，它默认具有内部链接。
C++标准，[namespace.unnamed],第1段,[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]阐述如下：
> 一个*匿名命名空间的定义*行为如下：

> inline namespace unique { /* empty body */}

> using namespace unique

> namespace unique { namespace-body }

> inline只出现在*匿名命名空间定义*中，翻译单元中所有出现的unique都被同一个标识符替代，这个标识符和整个程序中的其他标识符都不同。

生产质量的C++代码经常包含*头文件*，作为不同翻译单元间分享代码的方式。
头文件是通过#include指令被插入到一个翻译单元中的任何文件。
不要在头文件中定义匿名命名空间。
当在头文件中定义匿名命名空间时，会导致意想不到的结果。
由于匿名命名空间默认具有内部链接，每个翻译单元都会定义自己的唯一的匿名命名空间成员实例，这些实例在翻译单元中是[ODR使用](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-odr-use)的。
这样会导致非预期的结果，让可执行文件膨胀，或者由于破坏了单一定义规则(ODR)而意外触发[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，变量v被定义在位于头文件的一个匿名命名空间中，并可以从两个不同的翻译单元中访问。
每个翻译单元都会打印v的当前值并给它赋一个新值。
然而，由于v定义在匿名命名空间中，每个翻译单元都操作的是自己的v实例，导致非预期的输出。

{% highlight c++ %}
// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
namespace {
int v;
}
  
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
#include <iostream>
  
void f() {
  std::cout << "f(): " << v << std::endl;
  v = 42;
  // ...
}

// b.cpp
#include "a.h"
#include <iostream>
  
void g() {
  std::cout << "g(): " << v << std::endl;
  v = 100;
}
  
int main() {
  extern void f();
  f(); // Prints v, sets it to 42
  g(); // Prints v, sets it to 100
  f();
  g();
}
{% endhighlight %}

当程序执行的时候，输出如下：

```
f(): 0
g(): 0
f(): 42
g(): 100
```

# 遵从规范的解决方案

在这个遵从规范的解决方案中，v只被定义在一个翻译单元中，并且对所有的翻译单元是外部可见的，
产生预期的行为。

{% highlight c++ %}
// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
extern int v;
  
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
#include <iostream>
 
int v; // Definition of global variable v
  
void f() {
  std::cout << "f(): " << v << std::endl;
  v = 42;
  // ...
}
  
 // b.cpp
#include "a.h"
#include <iostream>
  
void g() {
  std::cout << "g(): " << v << std::endl;
  v = 100;
}
  
int main() {
  extern void f();
  f(); // Prints v, sets it to 42
  g(); // Prints v, sets it to 100
  f(); // Prints v, sets it back to 42
  g(); // Prints v, sets it back to 100
}

{% endhighlight %}

当程序执行时，输出如下：

```
f(): 0
g(): 42
f(): 100
g(): 42
```



# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，变量v被定义在一个头文件的匿名命名空间中，
同时一个inline函数get_v()也被定义了，该函数访问变量v。
从多个翻译单元中ODR式使用inline函数(就像f()和g()的实现)违反了[单一定义规则](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-onedefinitionrule)，
因为get_v()的定义在所有的翻译单元中并不相同，这是由于在每个翻译单元中它访问的是不同的v。

{% highlight c++ %}
// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
namespace {
int v;
}
  
inline int get_v() { return v; }
  
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
  
void f() {
  int i = get_v();
  // ...
}
  
// b.cpp
#include "a.h"
  
void g() {
  int i = get_v();
  // ...
}
{% endhighlight %}

参见[DCL60-CPP. Obey the one-definition rule](https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL60-CPP.+Obey+the+one-definition+rule)来查看有关违反单一定义规则的更多信息。

# 遵从规范的解决方案

在这个遵从规范的解决方案中，v只被定义在一个翻译单元中，但是对所有的翻译单元都外部可见，
而且可以从内联的get_v()函数进行访问。

{% highlight c++ %}
// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
extern int v;
 
inline int get_v() {
  return v;
}
 
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
  
// Externally used by get_v();
int v;
  
void f() {
  int i = get_v();
  // ...
}

// b.cpp
#include "a.h"
  
void g() {
  int i = get_v();
  // ...
}
{% endhighlight %}

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，函数f()被定义在头文件中。
然而，在多个翻译单元中包含这个头文件会违反单一定义规则，
并由于存在一个同名函数的多个定义导致链接时错误。

{% highlight c++ %}
// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
void f() { /* ... */ }
  
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
// ...
  
// b.cpp
#include "a.h"
// ...
{% endhighlight %}


# 不遵从规范的代码示例

这个不遵从规范的代码示例试图解决匿名命名空间中定义f()引起的链接时错误问题。
然而，它在最终的可执行文件中产生了多个不同定义的f()。
如果a.h被包含在多个翻译单元中，会导致链接时间增加，可执行文件变大，执行效率变低。

{% highlight c++ %}
// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
namespace { 
void f() { /* ... */ }
}
  
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
// ...
  
// b.cpp
#include "a.h"
// ...
{% endhighlight %}

# 遵从规范的解决方案

在这个遵从规范的解决方案中，f()没有被定义在匿名命名空间中，而是被定义成了内联函数。
内联函数要求在所有使用它的翻译单元中是相同的，这样允许[实现](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-implementation)上在运行时只产生该函数的唯一实例，函数体不在所有的调用点上都生成。

{% highlight c++ %}

// a.h
#ifndef A_HEADER_FILE
#define A_HEADER_FILE
  
inline void f() { /* ... */ }
  
#endif // A_HEADER_FILE
  
// a.cpp
#include "a.h"
// ...
  
// b.cpp
#include "a.h"
// ...
{% endhighlight %}

# 风险评估

在头文件中定义匿名命名空间会导致数据完整性破坏和性能问题，但是在不充足的测试下该问题经常很难被注意到。违反单一定义规则会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL59-CPP|中|不太可能|中|P4|L3|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682) |[DCL60-CPP. Obey the one-definition rule ](https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL60-CPP.+Obey+the+one-definition+rule)|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)|Subclause 3.2, "One Definition Rule"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 7.1.2, "Function Specifiers" |
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 7.3.1, "Namespace Definition" | 

# 参考链接

[DCL59-CPP. Do not define an unnamed namespace in a header file][1]

[template specialization][2]

[Definitions and ODR][3]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL59-CPP.+Do+not+define+an+unnamed+namespace+in+a+header+file

[2]: https://en.cppreference.com/w/cpp/language/template_specialization

[3]: https://en.cppreference.com/w/cpp/language/definition