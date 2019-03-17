---
layout: post
title: "DCL51-CPP 不要声明和定义保留标识符"
date:   2019-02-13 06:00:54 +0800
categories: jekyll update
---

C++标准，\[保留名字\][ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)中，规定了关于保留名字的以下规则：

> + 一个包含了标准库头文件的翻译单元，不应该#define或#undef标准库头文件中声明的名字
> + 一个翻译单元不应该#define和#undef与关键字、表3中列举的标识符、7.6节描述的attribute-token一样的名字
> + 包含两个下划线__的名字，下划线开头后面跟小写字母的名字都是实现某种功能的保留字
> + 以下划线开头的名字，是一种用来实现全局空间名字的保留字
> + 每个在头文件中声明为具有外部链接的对象的名字，都是用来保留实现这样的功能：给这个库文件对象指定外部链接。在std空间和全局空间都是如此. (???)
> + 每个在头文件中声明为具有外部链接的全局的函数签名，都是用来保留实现这样的功能：给这个函数签名指定外部链接(???)
> + 每个声明为具有外部链接的来自标准C库的名字，都是用来保留实现这样的功能：作为使用extern "C"链接的名字。在std空间和全局空间都是如此。
> + 每个声明为具有外部链接的来自标准C库的函数签名，都是用来保留实现这样的功能：作为使用extern "C"或者extern "C++"链接的函数签名，或者作为一个位于全局空间中的空间域的名字。（？？？ ）
> + 每个来自标准C库的类型T, 类型::T和std::T都是用来保留实现这样的功能：当::T定义了之后，::T和std::T是一致的。
> + 不是以下划线开头的字面后缀标识符(Literal suffix identifiers)用来保留作为未来使用。

在上面的标准摘录中所说的标识符和属性名字有: **override**, **final**, **alignas**, **carries_dependency**, **deprecated**和**noreturn**。

其他的标识符都不是保留的。在上下文中声明或定义保留的标识符会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。不要声明和定义一个保留的标识符。

# 不遵从规范的示例代码（头文件保护）

一种常用的实践是在预编译条件中使用宏，来防止对头文件的多重包含。虽然这种一种推荐做法，但是很多程序使用保留字来作为头文件防御的名字。这样的名字可能和C++标准模板库中的头文件中的定义名字发生冲突；即使未包含C++标准库头文件，也可能和编译器预先隐式定义的保留字相冲突。

{% highlight c++ %}
#ifndef _MY_HEADER_H_
#define _MY_HEADER_H_
 
// Contents of <my_header.h>
 
#endif // _MY_HEADER_H_
{% endhighlight %}

# 遵从规范的示例代码 (头文件保护)

下面的解决方案中，头文件保护定义的名字没有用下划线做开头或结尾

{% highlight c++ %}
#ifndef MY_HEADER_H
#define MY_HEADER_H
 
// Contents of <my_header.h>
 
#endif // MY_HEADER_H
{% endhighlight %}


# 不遵从规范的示例代码（[用户定义字面量](https://en.cppreference.com/w/cpp/language/user_literal)）

下面的不遵从规范的代码中，声明了一个用户定义字面量operator"" x。但是用户定义字面量的后缀要求以下划线开头。不以下划线开头的用户定义字面量后缀是保留给将来的库实现。

{% highlight c++ %}
#include <cstddef>

unsigned int operator"" x(const char *, std::size_t);
{% endhighlight %}


# 遵从规范的示例代码 （用户定义字面量）

下面的解决方案中，用户定义字面量是operator"" _x，不是一个保留标识符。

{% highlight c++ %}
#include <cstddef>

unsigned int operator"" _x(const char *, std::size_t);
{% endhighlight %}

用户定义字面量的名字是operator"" _x，而不是_x。 _x是一个全局空间的保留字。



# 不遵从规范的示例代码 （文件作用域对象）

在这个不遵从规范的示例代码中，文件作用域对象_max_limit和_limit的名字都是以下划线开头的。
由于_max_limit是static的，它看起来不会发生命名冲突。
但是，为了引入std::size_t而包含了头文件\<cstddef\>，所以潜在的命名冲突还是有可能存在。
(注意，即使没有包含任何C++标准模板库的头文件，一个遵从规范的编译器还是有可能隐式声明保留的名字)。
并且，由于_limit具有外部链接性，它有可能和运行库中的同名符号(symbol)相冲突，即使这个符号没有包含在任何头文件中。
所以，在任何文件作用域标识符的名字前添加下划线都是不安全的，即使这个标识符只对单个翻译单元可见。

{% highlight c++ %}
#include <cstddef> // std::for size_t

static const std::size_t _max_limit = 1024;
std::size_t _limit = 100;
 
unsigned int get_value(unsigned int count) {
  return count < _limit ? count : _limit;
}
{% endhighlight %}


# 遵从规范的示例代码 （文件作用域对象）

在这个遵从规范的解决方案中，文件作用域标识符都没有以下划线开头。

{% highlight c++ %}
#include <cstddef> // for size_t
 
static const std::size_t max_limit = 1024;
std::size_t limit = 100;
 
unsigned int get_value(unsigned int count) {
  return count < limit ? count : limit;
}
{% endhighlight %}


# 不遵从规范的示例代码 （保留的宏）

在这个不遵从规范的示例代码中，因为C++标准模板库头文件\<cinttypes\>被指定来包含\<cstdint\>，如\[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)\]中\[c.files\]的第4段所示,名字MAX_SIZE和\<cstdint\>头文件中的用来表示std::size_t上限的宏相冲突。

{% highlight c++ %}

#include <cinttypes> // for int_fast16_t
 
void f(std::int_fast16_t val) {
  enum { MAX_SIZE = 80 };
  // ...
}
{% endhighlight %}

# 遵从规范的示例代码 （保留的宏）

下面的解决方案中避免了重新定义保留的名字。

{% highlight c++ %}

#include <cinttypes> // for std::int_fast16_t
 
void f(std::int_fast16_t val) {
  enum { BufferSize = 80 };
  // ...
}
{% endhighlight %}


# 例外

**DCL51-CPP-EX1:**: 为了和一些其他编译器厂商或者语言标准模式兼容，创建一些和保留的标识符同样名字的宏标识符是可接受的，这样能实现行为在语义上相同，如下面例子所示： 

{% highlight c++ %}
// 有时由配置工具(如autoconf)产生
#define const const
  
// Allowed compilers with semantically equivalent extension behavior
#define inline __inline

{% endhighlight %}


**DCL51-CPP-EX2**: 如果你是编译器厂商或标准库开发者，那么使用保留的标识符是可以接受的。
保留的标识符可以被编译器定义，包含在标准库头文件中，或者标准库头文件所包含的头文件中，如下面例子所示。

{% highlight c++ %} 
// 下面声明的保留字存在于std::basic_string的libc++实现中。
// 原始代码可以在这里找到:
// http://llvm.org/svn/llvm-project/libcxx/trunk/include/string
template<class charT, class traits = char_traits<charT>, class Allocator = allocator<charT>>
class basic_string {
  // ...
  
  bool __invariants() const;
};
{% endhighlight %}

# 风险评估

使用保留的标识符会导致不正确的代码操作。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL51-CPP|低|不太可能|低|P3|L3|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[DCL58-CPP. Do not modify the standard namespaces](https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL58-CPP.+Do+not+modify+the+standard+namespaces)|
|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[DCL37-C. Do not declare or define a reserved identifier](https://wiki.sei.cmu.edu/confluence/display/c/DCL37-C.+Do+not+declare+or+define+a+reserved+identifier)|
|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[PRE06-C. Enclose header files in an include guard](https://wiki.sei.cmu.edu/confluence/display/c/PRE06-C.+Enclose+header+files+in+an+include+guard)|
|[MISRA C++:2008](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-MISRA08)|Rule 17-0-1|

# 参考书目

|\[[ISO/IEC 14882-2014\]](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 17.6.4.3, "Reserved Names"|
|\[[ISO/IEC 9899:2011\]](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO-IEC9899-2011)|Subclause 7.1.3, "Reserved Identifiers"|

# 参考链接

[DCL51-CPP. Do not declare or define a reserved identifier][1]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL51-CPP.+Do+not+declare+or+define+a+reserved+identifier
