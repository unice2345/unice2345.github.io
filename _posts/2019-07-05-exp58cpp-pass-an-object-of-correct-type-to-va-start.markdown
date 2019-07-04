---
layout: post
title:  "EXP58-CPP. 向va_start传入类型正确的对象"
date:   2019-07-05 05:26:54 +0800
categories: jekyll update
---

虽然规则[ DCL50-CPP. Do not define a C-style variadic function ](https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL50-CPP.+Do+not+define+a+C-style+variadic+function)禁止创建这样的函数，但是当函数具有C语言链接时它还是有可能被创建。
在这些条件下，当触发va_start()宏时要额外小心。
C标准库的宏va_start()对它的第二个参数值的类型有几个语义限制。
C标准，第7.16.1.4节，第4段 [[ISO/IEC 9899:2011](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO-IEC9899-2011)]阐述如下：

> The parameter parmN is the identifier of the rightmost parameter in the variable parameter list in the function definition (the one just before the ...). If the parameter parmN is declared with the register storage class, with a function or array type, or with a type that is not compatible with the type that results after application of the default argument promotions, the behavior is undefined.

这些限制被C++标准,[support.runtime],第3段, [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]取代，阐述如下：

> The restrictions that ISO C places on the second parameter to the va_start() macro in header <stdarg.h> are different in this International Standard. The parameter parmN is the identifier of the rightmost parameter in the variable parameter list of the function definition (the one just before the ...). If the parameter parmN is of a reference type, or of a type that is not compatible with the type that results when passing an argument for which there is no parameter, the behavior is undefined.

语义要求的主要区别是:

+ 不允许向va_start()的第二个参数传入引用类型.
+ 向va_start()传入的第二个参数的对象的类类型具有非平凡的拷贝构造函数，非平凡的移动构造函数或者非平凡的构造函数，是被有条件支持的，是由实现定义的语义([expr.call]第7段).
+ 可以传入一个用register关键字声明的参数([dcl.stc]第3段)或函数类型的参数。

传入一个array类型的对象还是会引起[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)，因为array类型作为函数参数需要使用引用，这是被禁止的。
并且，传入一个会进行默认参数提升的对象类型在C++中仍然会导致未定义的行为。

# 不符合规范的代码示例

在这个不符合规范的代码示例中，传入va_start()的对象会进行默认参数提升，会导致未定义的行为。

{% highlight c++ %}
#include <cstdarg>
  
extern "C" void f(float a, ...) {
  va_list list;
  va_start(list, a);
  // ...
  va_end(list);
}
{% endhighlight %}

# 符合规范的解决方案

在这个符合规范的解决方案中，f()接受了double而非float。

{% highlight c++ %}
#include <cstdarg>
  
extern "C" void f(double a, ...) {
  va_list list;
  va_start(list, a);
  // ...
  va_end(list);
}
{% endhighlight %}

# 不符合规范的代码示例

在这个不符合规范的代码示例中，一个引用类型被传入va_start()的第二个参数。

{% highlight c++ %}

#include <cstdarg>
#include <iostream>
  
extern "C" void f(int &a, ...) {
  va_list list;
  va_start(list, a);
  if (a) {
    std::cout << a << ", " << va_arg(list, int);
    a = 100; // Assign something to a for the caller
  }
  va_end(list);
}
{% endhighlight %}

# 符合规范的解决方案

没有给f()传入引用类型，这个符合规范的解决方案传入的是指针类型。（？？？执行也有异常？？？）

{% highlight c++ %}
#include <cstdarg>
#include <iostream>
  
extern "C" void f(int *a, ...) {
  va_list list;
  va_start(list, a);
  if (a && *a) {
    std::cout << a << ", " << va_arg(list, int);
    *a = 100; // Assign something to *a for the caller
  }
  va_end(list);
}
{% endhighlight %}


# 不符合规范的代码示例

在这个不符合规范的代码示例中，一个具有非平凡拷贝构造函数(std::string)的类被传入va_start()作为第二个参数，这是被有条件支持的，取决于[实现](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-implementation).

{% highlight c++ %}
#include <cstdarg>
#include <iostream>
#include <string>
  
extern "C" void f(std::string s, ...) {
  va_list list;
  va_start(list, s);
  std::cout << s << ", " << va_arg(list, int);
  va_end(list);
}
{% endhighlight %}

# 符合规范的解决方案

这个符合规范的解决方案传入了const char*而非std::string，在任何实现上都是定义良好的。

{% highlight c++ %}

#include <cstdarg>
#include <iostream>
  
extern "C" void f(const char *s, ...) {
  va_list list;
  va_start(list, s);
  std::cout << (s ? s : "") << ", " << va_arg(list, int);
  va_end(list);
}
{% endhighlight %}


# 风险评估

向va_start()传入一个不支持的类型的对象作为第二个参数会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior),有可能被[利用](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-exploit)，导致数据完整性被破坏。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP58-CPP|中|不太可能|中|P4|L3|

# 相关规则

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[DCL50-CPP. Do not define a C-style variadic function](https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL50-CPP.+Do+not+define+a+C-style+variadic+function)|

# 参考书目

|[ISO/IEC 9899:2011](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO-IEC9899-2011)|Subclause 7.16.1.4, "The va_start Macro"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 18.10, "Other Runtime Support"|

# 参考

[EXP58-CPP. Pass an object of the correct type to va_start][1]

[va_start][2]

[va_arg][3]

[variadic_arguments#Default_conversions][4]

[Trivial default constructor][5]

[Trivial copy constructor][6]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP58-CPP.+Pass+an+object+of+the+correct+type+to+va_start

[2]: https://en.cppreference.com/w/cpp/utility/variadic/va_start

[3]: https://en.cppreference.com/w/cpp/utility/variadic/va_arg

[4]: https://en.cppreference.com/w/cpp/language/variadic_arguments#Default_conversions

[5]: https://en.cppreference.com/w/cpp/language/default_constructor

[6]: https://en.cppreference.com/w/cpp/language/copy_constructor

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。