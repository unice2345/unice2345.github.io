---
layout: post
title:  "DCL60-CPP 遵守单一定义规则"
date:   2019-04-10 06:26:54 +0800
categories: jekyll update
---

大型的C++程序通常被分成多个翻译单元，这些翻译单元之后被链接到一起组成一个可执行程序。
为了支持这个模型，C++限制了命名的对象的定义，要求一个对象在所有的翻译单元中只能有一个定义，
来保证链接的行为是确定性的。
这个模型被称为*单一定义规则*(ODR)，在C++标准，[basic.def.odr]的第4段中有定义[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]。

每个程序中只能包含非内联函数或ODR使用的变量的一个定义，不需要诊断信息(???)。
定义可以显示出现在程序中，可以存在于标准库或用户自定义库中，或者在合适的时候被隐式地定义。
一个内联函数应该被定义在它被ODR使用的每个翻译单元中。

多个翻译单元进行编译最常用的方法是在一个头文件中进行声明，然后源文件通过#include可以访问头文件中的声明。这些声明通常也是定义，比如像类和函数模板。这个方法作为一个例外是被允许的，在第6段中部分阐述如下：
> 对于以下类型在程序中可以有多于一个定义：类，枚举，具有外部链接的内联函数，类模板，非静态函数模板，类模板的静态数据成员，类模板的成员函数或者有些模板参数未被指定的模板特化，但要满足如下条件：每个定义存在于不同的翻译单元，并且定义满足如下条件：比如一个名字为D的实体被定义在多于一个的翻译单元中...

> 如果D的定义满足上面所有的条件，那么程序会表现为只有一个D的定义。如果D不满足上面所有的条件，行为是未定义的。

第6段中要求的条件本质上是说两个定义必须完全相同(而不是简单的相等)。
结果是，在两个不同的翻译单元中通#include指令引入的定义一般不会破坏单一定义规则，
因为定义在两个翻译单元中是相同的。

然而，在使用块语言链接规范，厂商特定的语言扩展等情况下，通过引入#include的定义有可能破坏单一定义规则。破坏单一定义规则的一种更可能的场景是，不小心在不同的翻译单元中给不同的对象进行相同的定义。

不要违反单一定义规则，否则会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，两个不同的翻译单元定义了同名的类，但它们的定义却不相同。
虽然这两个类的功能相同(它们都定义了类S，并包含一个公有的非静态的数据成员int a)，
但它们的定义并不具有同样的符号序列(???)。
这个代码示例违反了ODR，会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}

// a.cpp
struct S {
  int a;
};
  
// b.cpp
class S {
public:
  int a;
};
{% endhighlight %}

# 遵从规范的解决方案

正确的解决措施取决于程序员的意图。如果程序员是为了使一个相同的类定义对两个翻译单元都可见，
解决方法是使用一个头文件，将对象同时引入两个翻译单元中，如下面的解决方案所示：

{% highlight c++ %}

// S.h
struct S {
  int a;
};
  
// a.cpp
#include "S.h"
  
// b.cpp
#include "S.h"
{% endhighlight %}

# 遵从规范的解决方案

如果是由于不经意的命名冲突导致的违反ODR，最好的解决方案是保证两个类的定义是唯一的，就像下面的解决方案所示：

{% highlight c++ %}
// a.cpp
namespace {
struct S {
  int a;
};
}
  
// b.cpp
namespace {
class S {
public:
  int a;
};
}
{% endhighlight %}

或者，在每个翻译单元中给类指定不同的名字来避免违反ODR。

# 不遵从规范的代码示例(Microsoft Visual Studio)

在这个不遵从规范的代码示例中，一个类定义通过使用#include被引入到两个翻译单元中。
然而，其中一个翻译单元使用了一个[实现定义的](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-implementation-definedbed)#pragma (这里是Microsoft Visual Studio)来指定结构体字段的对齐要求。
结果是，两个类定义在各自的翻译单元中可能具有不同的内存布局，导致违反ODR。

{% highlight c++ %}
// s.h
struct S {
  char c;
  int a;
};
  
void init_s(S &s);
  
// s.cpp
#include "s.h"
  
void init_s(S &s); {
  s.c = 'a';
  s.a = 12;
}
  
// a.cpp
#pragma pack(push, 1)
#include "s.h"
#pragma pack(pop)
  
void f() {
  S s;
  init_s(s);
}
{% endhighlight %}


**实现细节**

在上面的不遵从规范的代码示例中，是有可能导致a.cpp中通过init_s()为对象分配的空间大小和s.cpp中分配的空间大小是不同的。
在翻译s.cpp时，数据结构的内存布局，可能在c和a数据成员之间添加填充字节。
在翻译a.cpp时，由于使用了#pragma pack指令，数据结构的内存布局，可能删除了这些填充字节，
所以传入init_s()的对象可能比预想的要小。
结果是，当init_s()初始化s的数据成员时，可能会导致buffer越界。

# 遵从规范的解决方案

在这个遵从规范的解决方案中，由实现定义的数据结构成员对齐指令被删除了，
保证S所有的定义都遵从ODR。

{% highlight c++ %}
// s.h
struct S {
  char c;
  int a;
};
  
void init_s(S &s);
  
// s.cpp
#include "s.h"
  
void init_s(S &s); {
  s.c = 'a';
  s.a = 12;
}
  
// a.cpp
#include "s.h"
  
void f() {
  S s;
  init_s(s);
}
{% endhighlight %}

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，常量对象n具有内部链接，但是在f()中被ord-used了，而f()具有外部链接。由于f()被声明为内联函数，f()在所有翻译单元中的定义必须相同。
然而，每个翻译单元都有各自唯一的n实例，导致违反ODR。

{% highlight c++ %}
const int n = 42;
  
int g(const int &lhs, const int &rhs);
  
inline int f(int k) {
  return g(k, n);
}
{% endhighlight %}

# 遵从规范的解决方案

一个遵从规范的解决方案必须改变下面三个因素之一：(1) 在f()中不能ODR使用n; (2) 必须将n声明为具有外部链接性； (3) 不能将函数f()定义为内联函数.

如果条件允许改变函数g()的签名，使其可以接受按值传递的参数而不是按引用传递，那么f()中的n就不再是ODR使用的，因为n会被评估为一个常量表达式。
这个解决方案是符合规范的，但并不理想。可能有时无法修改g()的签名，比如g()如果是\<algorithm\>中的std::max()。
而且由于n和f()具有不同的链接性，当f()被修改成ODR使用n时，违反ODR还是会发生。

{% highlight c++ %}

const int n = 42;
  
int g(int lhs, int rhs);
  
inline int f(int k) {
  return g(k, n);
}
{% endhighlight %}

# 遵从规范的解决方案

在这个遵从规范的解决方案中，常量对象n被替换成同名的枚举量(enumerator)。
被定义在命名空间的枚举和它所处的命名空间具有同样的链接性。
全局命名空间具有外部链接性，所以枚举的定义和它内部包含的枚举量也具有外部链接性。
虽然看起来不那么美观，但是这个符合规范的解决方案没有上面代码的维护负担，因为n和f()具有同样的链接性。

{% highlight c++ %}

enum Constants {
  N = 42
};
 
int g(const int &lhs, const int &rhs);
  
inline int f(int k) {
  return g(k, N);
}
{% endhighlight %}

# 风险评估

违反ODR会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior), 进一步导致漏洞利用，如拒绝服务攻击.
如"Support for Whole-Program Analysis and the Verification of the One-Definition Rule in C++" [[Quinlan 06](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Quinlan06)]展示的那样，没有强制遵守ODR会导致名为*VPTR*[漏洞利用](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-exploit)的虚函数指针攻击。
在这个漏洞利用中，一个对象的虚函数表被破坏，所以调用这个对象的虚函数会导致恶意代码被执行。
参见Quinlan和其同事的论文来了解更详细的信息。
然而，需要注意的是为了引入恶意的类，攻击者必须获取系统构建代码的能力。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL60-CPP|高|不太可能|高|P3|L3|

**自动检测**

略

**相关漏洞**

略

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 3.2, "One Definition Rule"|
|[[Quinlan 2006](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Quinlan06)]||

# 参考链接

[DCL60-CPP. Obey the one-definition rule][1]

[Language linkage][2]

[Storage Classes and Linkage][3]

[Storage class specifiers][4]


[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL60-CPP.+Obey+the+one-definition+rule

[2]: https://en.cppreference.com/w/cpp/language/language_linkage

[3]: http://cs.stmarys.ca/~porter/csc/common_341_342/notes/storage_linkage.html

[4]: https://en.cppreference.com/w/cpp/language/storage_duration