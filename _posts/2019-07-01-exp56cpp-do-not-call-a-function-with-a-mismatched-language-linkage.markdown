---
layout: post
title:  "EXP56-CPP 不要用不匹配的语言链接调用函数"
date:   2019-7-1 06:26:54 +0800
categories: jekyll update
---

C++允许通过使用语言链接规范和其他编程语言进行一定程度的互操作。
这些规范影响了函数被调用或者数据被访问的方式。
默认的，所有函数类型，以及函数和变量名字都具有C++外部链接，虽然可以指定其他语言外部链接。
C++标准要求实现支持C和C++语言链接，但其他语言的链接是实现自己定义的语义，如java, Ada和FORTRAN.

语言链接被指定为函数类型的一部分，根据C++标准，[dcl.link],第1段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)], 描述如下：

> Two function types with different language linkages are distinct types even if they are otherwise identical.

当调用一个函数时，如果调用的函数类型的语言链接和函数定义的语言链接不匹配，是[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。
比如，一个不匹配的语言链接会破坏调用栈，由于调用方式或其他ABI不匹配。

不要调用一个和函数定义语言链接类型不匹配的函数。
这个限制适用于C++语言中的函数，也适用于从C++外部调用的函数指针。

然而，很多编译器没有将语言链接实现为函数类型的一部分，尽管C++标准要求这样做。For instance, GCC 6.1.0, Clang 3.9, and Microsoft Visual Studio 2015 all consider the following code snippet to be ill-formed due to a redefinition of f() rather than a well-formed overload of f().

{% highlight c++ %}
typedef void (*cpp_func)(void);
extern "C" typedef void (*c_func)(void);
 
void f(cpp_func fp) {}
void f(c_func fp) {}
{% endhighlight %}

Some compilers conform to the C++ Standard, but only in their strictest conformance mode, such as EDG 4.11. This implementation divergence from the C++ Standard is a matter of practical design trade-offs. Compilers are required to support only the "C" and "C++" language linkages, and interoperability between these two languages often does not require significant code generation differences beyond the mangling of function types for most common architectures such as x86, x86-64, and ARM. There are extant Standard Template Library implementations for which language linkage specifications being correctly implemented as part of the function type would break existing code on common platforms where the language linkage has no effect on the runtime implementation of a function call.

当语言链接规格、运行时平台和编译器实现的结合不会对函数调用的运行时行为造成影响时，调用不匹配的语言链接的函数是允许的。
比如，下面的代码在x86下的Microsoft Visual Studio 2015是可以编译的，
虽然lamda函数被隐式转换为具有C++链接的函数指针类型，而qsort()接受的是C链接的函数指针类型。

{% highlight c++ %}

#include <cstdlib>
  
void f(int *int_list, size_t count) {
  std::qsort(int_list, count, sizeof(int),
             [](const void *lhs, const void *rhs) -> int {
               return reinterpret_cast<const int *>(lhs) <
                      reinterpret_cast<const int *>(rhs);
             });
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，call_java_fn_ptr()期望接受一个具有java语言链接的函数指针，
这个函数指针会被Java解释器使用并被C++代码回调。
然而，该函数被给与了C++语言链接的函数指针，当解释器试图调用函数指针时会导致未定义的行为。
这段代码是病态的(ill-formed)因为callback_func()和java_callback的类型不同。
However, due to common implementation divergence from the C++ Standard, some compilers may incorrectly accept this code without issuing a diagnostic.

{% highlight c++ %}

extern "java" typedef void (*java_callback)(int);
  
extern void call_java_fn_ptr(java_callback callback);
void callback_func(int);
  
void f() {
  call_java_fn_ptr(callback_func);
}
{% endhighlight %}

# 符合规范的解决方案

在这个符合规范的解决方案中，callback_func()被给与了一个“java”语言链接
和java_callback的语言链接类型匹配。

{% highlight c++ %}
extern "java" typedef void (*java_callback)(int);
 
extern void call_java_fn_ptr(java_callback callback);
extern "java" void callback_func(int);
 
void f() {
  call_java_fn_ptr(callback_func);
}
{% endhighlight %}

# 风险评估

不匹配的语言链接规格在C和C++语言链接时一般不会造成可利用的安全漏洞。
然而，其他语言链接存在着未定义行为，更容易造成程序非正常执行，包括可利用的漏洞等问题。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP56-CPP|低|不太可能|中|P2|L3|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 5.2.2, "Function Call"|
||Subclause 7.5, "Linkage Specifications" |

# 参考

[EXP56-CPP. Do not call a function with a mismatched language linkage][1]

[Language linkage][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP56-CPP.+Do+not+call+a+function+with+a+mismatched+language+linkage

[2]: https://en.cppreference.com/w/cpp/language/language_linkage


<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。