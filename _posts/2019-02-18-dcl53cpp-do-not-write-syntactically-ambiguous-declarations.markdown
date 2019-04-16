---
layout: post
title:  "DCL53-CPP 不要写语法上有歧义的声明"
date:   2019-02-18 07:16:54 +0800
categories: jekyll update
---

有可能会写出有歧义的语法，既可以解释为一个表达式语句，或者解释为一个声明。
这类语法称之为*令人费解的解析(vexing parse)*，因为编译器必须采取消歧义的规则来决定语义上的结果。
C++标准，\[stmt.ambig\],第一段 \[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)\]中部分如下阐述：

> 在语法上，表达式语句和声明可能会造成歧义：一个表达式语句如果它将函数风格的显示类型转换为最左边的子表达式，那么，它和一个由(开头的声明是有歧义的。在这种情况下，语句就是一个声明。\[注意：为了消除歧义，需要检查整个表达式来决定它到底是一个表达式语句还是一个声明。\]

另外一种类型的令人费解的解析存在于声明时，一个语法既可以被认为是一个函数声明，也可以被认为是具有函数风格转换的初始化参数的声明(a declaration with a function-style cast as the initializer???)。 C++标准，\[dcl.ambig.res\], 第1段，部分阐述如下：

> 这种歧义性来自于函数风格的类型转换和第6.8节中提到的声明的相似性，这种歧义性会发生在声明的情景下。在这种情景下，一个参数周围有一系列冗余的括号的函数声明，和一个以函数风格类型转换初始化的对象声明，这两者之间需要做一个选择。就像第6.8节中提到的歧义性一样，任意一种认为构建可以成为一个声明的解析就是一个声明( the resolution is to consider any construct that could possibly be a declaration a declaration.???)。

不要写语法上具有歧义的声明。随着使用列表初始化的统一的声明语法的到来，现在有无歧义的语法来指定一个声明而非表达式语句。
声明也可以采用以下方法来消除歧义，使用非函数风格的类型转换，使用=，或者删除参数周围不相干的括号。

# 不遵从规范的示例代码

在下面这个不遵从规范的示例代码中，声明了一个造成歧义的std::unique_lock类型的临时变量，预想利用[RAII](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-RAII)功能去加锁和解锁mutex m.
但是，这个声明在语法上具有歧义性，它既可以被解释为声明了一个调用单个参数转换构造器的匿名对象，
也可以被解释为利用默认构造函数声明了一个名字为m的对象(参见[direct initialization][4])。
在这个例子中使用的语法实际上是后一种情形而非前一种情形，所以mutex对象永远都不会被加锁。

{% highlight c++ %}
#include <mutex>

static std::mutex m;
static int shared_resource;

void increment_by_42()
{
    std::unique_lock<std::mutex>(m);
    shared_resource += 42;
}
{% endhighlight %}


# 遵从规范的示例代码

在这个遵从规范的示例代码中，lock对象被指定了一个不同于m的标识符，这样合适的转换构造器就会被调用。

{% highlight c++ %}
#include <mutex>
  
static std::mutex m;
static int shared_resource;
 
void increment_by_42() {
  std::unique_lock<std::mutex> lock(m);
  shared_resource += 42;
}
{% endhighlight %}

# 不遵从规范的示例代码

在这个不遵从规范的示例代码中，试图去声明一个Widge类型的临时变量w，采用默认构造函数。
然而，这个声明在语法上是有歧义的，这段代码可以被认为是声明了一个没有入参并且返回值是Widget的函数指针，也可以认为是一个Widget类型的临时变量声明。例子中的这个语法解析成前者而非后者。

{% highlight c++ %}
#include <iostream>
  
struct Widget {
  Widget() { std::cout << "Constructed" << std::endl; }
};
 
void f() {
  Widget w();
}
{% endhighlight %}

结果就是，程序编译执行后没有任何输出，因为默认构造函数没有被调用。

# 遵从规范的示例代码

这个遵从规范的解决方案展示了两种合规的写声明的方式。
第一种方式是去掉临时变量名字后的括号，这样在语法上保证是一个变量声明而非函数声明。
第二种方式是使用*花括号初始化列表*([list_initialization][5])来直接声明临时变量。

{% highlight c++ %}
#include <iostream>
  
struct Widget {
  Widget() { std::cout << "Constructed" << std::endl; }
};
 
void f() {
  Widget w1; // Elide the parentheses
  Widget w2{}; // Use direct initialization
}
{% endhighlight %}

执行程序后会输出两次Constructed，一次是w1输出的，一次是w2输出的。

# 不遵从规范的示例代码

下面这个不遵从规范的示例代码展示了令人费解的解析。
声明Gadget g(Widget(i));没有被解析成具有单个参数的Gadget对象。
它被解析成了一个函数声明，并且其参数周围有冗余的括号。


{% highlight c++ %}
#include <iostream>
 
struct Widget {
  explicit Widget(int i) { std::cout << "Widget constructed" << std::endl; }
};
 
struct Gadget {
  explicit Gadget(Widget wid) { std::cout << "Gadget constructed" << std::endl; }
};
 
void f() {
  int i = 3;
  Gadget g(Widget(i));
  std::cout << i << std::endl;
}
{% endhighlight %}

参数周围的括号是可有可无的，所以下面这句声明和Gadget g(Widget(i));在语义是一样的。

{% highlight c++ %}
Gadget g(Widget i);
{% endhighlight %}

所以，这个程序是可以编译通过的，并且只输出了3，因为Gadget和Widget对象都没有被构建。


# 遵从规范的示例代码

下面的遵从规范的示例代码展示了两种同样效果的符合规范的声明g的方法。
第一个声明g1,在构造函数的参数周围使用了额外的括号，强制使编译器解析成一个Gadget类型的临时变量声明，而非函数声明。
第二个声明g2，使用直接初始化达到类型的效果。

{% highlight c++ %}
#include <iostream>
 
struct Widget {
  explicit Widget(int i) { std::cout << "Widget constructed" << std::endl; }
};
 
struct Gadget {
  explicit Gadget(Widget wid) { std::cout << "Gadget constructed" << std::endl; }
};
 
void f() {
  int i = 3;
  Gadget g1((Widget(i))); // Use extra parentheses
  Gadget g2{Widget(i)}; // Use direct initialization
  std::cout << i << std::endl;
}
{% endhighlight %}

执行程序会产生符合预期的输出：

{% highlight c++ %}
Widget constructed
Gadget constructed
Widget constructed
Gadget constructed
3
{% endhighlight %}

# 风险评估

语法上有歧义的声明会导致非预期的程序执行。然而，一般来说基本的测试就可以发现破坏这个规则的行为。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL53-CPP|低|不太可能|中|P2|L3|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 6.8, "Ambiguity Resolution"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 8.2, "Ambiguity Resolution"|
|[Meyers 2001](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Meyers01)|Item 6, "Be Alert for C++'s Most Vexing Parse"|


# 参考链接

[DCL53-CPP. Do not write syntactically ambiguous declarations][1]

[explicit限定符][2]

[converting constructor][3]

[direct initialization][4]

[list_initialization][5]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL53-CPP.+Do+not+write+syntactically+ambiguous+declarations
[2]: https://en.cppreference.com/w/cpp/language/explicit
[3]: https://en.cppreference.com/w/cpp/language/converting_constructor
[4]: https://en.cppreference.com/w/cpp/language/direct_initialization
[5]: https://en.cppreference.com/w/cpp/language/list_initialization

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。