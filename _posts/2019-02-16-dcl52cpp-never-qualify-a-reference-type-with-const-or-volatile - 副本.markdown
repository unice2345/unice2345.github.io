---
layout: post
title:  "DCL52-CPP 不要用const或volatile修饰引用类型"
date:   2019-02-16 08:26:54 +0800
categories: jekyll update
---

C++不允许改变引用类型的类型值，相当于对将所有的引用都用const修饰。C++标准\[dcl.ref\]第1段\[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)\]阐述如下：

> cv限定符修饰的引用是不正确的，通过使用typedef-name(7.1.3,14.1)或dectype-specifier引入的cv限定符除外，这两种情况下cv限定符会被忽略。

所以，C++禁止或忽略引用类型的[cv限定符](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-cvqualify)。只有非引用类型可以被cv限定符修饰。

当在引用类型中试图使用const修饰符时，程序员很容易写成下面这样的代码：

{% highlight c++ %}
char &const p;
{% endhighlight %}

而不是这样的代码：

{% highlight c++ %}
char const &p; // 或者const char& p;
{% endhighlight %}

不要用cv限定符去修饰引用类型，这样会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。
[符合规范的](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-conformingprogram)编译器会对此发出[诊断信息](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-diagnosticmessage)。
但是，如果编译器没有发出[致命诊断信息](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-fataldiagnostic),程序可能会产生奇怪的结果，
比如p所指向的字符变量被修改。

# 不遵从规范的示例代码

在下面这个不遵从规范的示例代码中，创建了一个指向char的用const修饰的引用，而不是一个指向用const修饰的char的引用。 这样会导致未定义的行为。

{% highlight c++ %}
#include <iostream>
 
void f(char c) {
  char &const p = c;
  p = 'p';
  std::cout << c << std::endl;
}
{% endhighlight %}

**实现细节(MSVC)**

在[Microsoft Visio Studio](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-msvc) 2015中，这段代码能够编译通过，但会产生如下警告信息:

> warning C4227: anachronism used : qualifiers on reference are ignored

程序运行时，输出结果：

> p

**实现细节(Clang)**

在[Clang](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-clang) 3.9中，这段代码产生致命信息：

> error: 'const' qualifier may not be applied to a reference

# 不遵从规范的示例代码

下面这个不遵从规范的代码，正确地将p声明为一个指向由const的修饰的char变量的引用。
但是后面修改p的值导致程序[不正确](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-ill-formed)。

{% highlight c++ %}
#include <iostream>
 
void f(char c) {
  const char &p = c;
  p = 'p'; // 错误: 只读变量不能被赋值
  std::cout << c << std::endl;
}
{% endhighlight %}

# 遵从规范的解决方案

下面这个遵从规范的解决方案删除了const限定符。

{% highlight c++ %}
#include <iostream>
 
void f(char c) {
  char &p = c;
  p = 'p';
  std::cout << c << std::endl;
}
{% endhighlight %}

# 风险评估

由const或volatile修饰引用类型时，如果不产生致命信息，就会导致未定义的行为，引起非预期的值被存储，以及可能破坏数据完整性。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL52-CPP|低|不太可能|低|P3|L3|

**自动检测**

略

**相关漏洞**

略

# 参考书目

|\[[Dewhurst 2002](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Dewhurst02)\]|Gotcha #5, "Misunderstanding References"|
|\[[ISO/IEC 14882-2014\]](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 8.3.2, "References"|

# 参考链接

[DCL52-CPP. Never qualify a reference type with const or volatile][1]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL52-CPP.+Never+qualify+a+reference+type+with+const+or+volatile

