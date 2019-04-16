---
layout: post
title:  "DCL58-CPP. 不要修改标准命名空间"
date:   2019-03-25 07:24:54 +0800
categories: jekyll update
---

命名空间为声明引入了新的声明区域，降低了同其他声明区域中的标识符的重名概率。
命名空间的一个特性是它可以被扩展，甚至在不同的翻译单元中。
比如，下面的声明是合法的：

{% highlight c++ %}
namespace MyNamespace {
int i;
}
  
namespace MyNamespace {
int i;
}
  
void f() {
  MyNamespace::i = MyNamespace::i = 12;
}
{% endhighlight %}

标准库引入了命名空间std，提供了一些标准的声明，比如std::string, std::vector和std::for_each等。然而，除了在一些特定的条件下，向标准库中引入新的声明会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).
C++标准,[namespace.std],第1段和第2段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]，阐述如下：
> 1.如果一个C++程序向std命名空间或者std内部的命名空间添加声明或定义，其行为是未定义的。
一个程序向任何std命名空间中的标准库模板添加模板特化时，只有声明取决于用户定义类型并且特化满足标准库对原始模板的要求时，这种添加行为才不被显式禁止。

> 2.一个C++程序的行为是未定义的，如果它声明了:

>  --- 对一个标准库类模板的任何成员函数的显式特化，或者

>  --- 对一个标准库类或类模板的任何成员函数模板的显式特化，或者

>  --- 对一个标准库类或类模板的任何成员类模板的显式或部分特化。

除了对std命名空间的扩展限制，C++标准，[namespace.posix]，第1段，进一步阐述如下：
> 一个C++程序的行为是未定义的，如果它向posix命名空间或者posix命名空间内部的命名空间添加声明或定义，除非特殊指定。posix命名空间是为ISO/IEC 9945和其他POSIX标准保留的。

不要向标准命名空间std或posix或它们内部包含的命名空间中添加声明或定义，除非是添加一个取决于用户定义类型并且满足标准库对于原始模板要求的模板特化。

标准库工作组(The Library Working Group)，负责C++标准中标准库章节的起草，在对*用户自定义类型*的定义上有一个未解决的[问题](http://www.open-std.org/jtc1/sc22/wg21/docs/lwg-active.html#2139)。尽管标准库工作组在定义上没有官方立场 [[INCITS 2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-INCITS2014)]，我们在这里将其定义为任何没有包含在std命名空间或std命名空间内部命名空间中的任何class, struct, union或者enum。等同的，它是用户提供的类型，而非标准库提供的类型。

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，声明x被添加到命名空间std中，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).

{% highlight c++ %}
namespace std {
int x;
}
{% endhighlight %}

# 遵从规范的解决方案

这个遵从规范的解决方案假定程序员的意图是将声明x放置到一个命名空间中，为了防止和其他全局的标识符发生冲突。不将声明放置到命名空间std中，而是将其放入一个没有保留名的命名空间中。

{% highlight c++ %}
namespace nonstd {
int x;
}
{% endhighlight %}

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，一个对std::plus特化的模板被添加到std命名空间，为了实现std::plus可以连接std::string和MyString对象。然而，因为模板特化的是标准库提供的类型(std::string)，这段代码会导致未定义的行为。

{% highlight c++ %}
#include <functional>
#include <iostream>
#include <string>

class MyString {
  std::string data;
  
public:
  MyString(const std::string &data) : data(data) {}
  
  const std::string &get_data() const { return data; }
};

namespace std {
template <>
struct plus<string> : binary_function<string, MyString, string> {
  string operator()(const string &lhs, const MyString &rhs) const {
    return lhs + rhs.get_data();
  }
};
}

void f() {
  std::string s1("My String");
  MyString s2(" + Your String");
  std::plus<std::string> p;
  
  std::cout << p(s1, s2) << std::endl;
}
{% endhighlight %}

# 遵从规范的解决方案

std::plus的接口要求函数调用操作符的两个入参和返回值是同一类型。
因为在不遵从规范的代码示例中尝试的特化会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior),
这个遵从规范的解决方案定义了一个新的std::bianry_function子类，可以把std::string加到MyString对象上，而不会修改std命名空间。

{% highlight c++ %}
#include <functional>
#include <iostream>
#include <string>

class MyString {
  std::string data;
  
public:
  MyString(const std::string &data) : data(data) {}
  
  const std::string &get_data() const { return data; }
};

struct my_plus : std::binary_function<std::string, MyString, std::string> {
  std::string operator()(const std::string &lhs, const MyString &rhs) const {
    return lhs + rhs.get_data();
  }
};

void f() {
  std::string s1("My String");
  MyString s2(" + Your String");
  my_plus p;
  
  std::cout << p(s1, s2) << std::endl;
}
{% endhighlight %}

# 遵从规范的解决方案

在这个遵从规范的解决方案中， 一个对std::plus的模板特化被添加到std命名空间中，
但是模板特化取决于用户定义的类型，满足标准模板库对原始模板的要求，所以它符合规则。
然而，由于MyString可以从std::string构建，这个符合规范的解决方案触发了一个转换构造器，
这在前面的解决方案中是不存在的。

{% highlight c++ %}
#include <functional>
#include <iostream>
#include <string>
 
class MyString {
  std::string data;
   
public:
  MyString(const std::string &data) : data(data) {}
   
  const std::string &get_data() const { return data; }
};
 
namespace std {
template <>
struct plus<MyString> {
  MyString operator()(const MyString &lhs, const MyString &rhs) const {
    return lhs.get_data() + rhs.get_data();
  }
};
}
 
void f() {
  std::string s1("My String");
  MyString s2(" + Your String");
  std::plus<MyString> p;

  std::cout << p(s1, s2).get_data() << std::endl;
}
{% endhighlight %}

# 风险评估

在C++标准模板库中修改标准命名空间会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL58-CPP|高|不太可能|中|P6|L2|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682) |[DCL51-CPP. Do not declare or define a reserved identifier](https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL51-CPP.+Do+not+declare+or+define+a+reserved+identifier)|

# 参考书目

|[INCITS 2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-INCITS2014)|Issue 2139, "What Is a User-Defined Type?"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)|Subclause 17.6.4.2.1, "Namespace std"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 17.6.4.2.2, "Namespace posix" |

# 参考链接

[DCL58-CPP. Do not modify the standard namespaces][1]

[template specialization][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL58-CPP.+Do+not+modify+the+standard+namespaces

[2]: https://en.cppreference.com/w/cpp/language/template_specialization

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。