---
layout: post
title:  "EXP52-CPP 不要依赖未求值操作数的副作用"
date:   2019-05-06 06:10:54 +0800
categories: jekyll update
---

一些和操作数相关的表达式是*未求值的*。
C++标准，[expr.expr], 第8段 [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)], 阐述如下：

> 在一些场景下，会出现*未求值的操作数*。
一个未求值的操作数是没有被求值的。
未求值的操作数被认为是全表达式。
[注意：在一个未求值的操作数中，一个非静态的类成员可能被命名(5.1)，而且对象或函数的名字本身不需要提供定义。]

下面这些表达式不对它们的操作数进行求值：sizeof(), typeid(), noexcept(), decltype()和declval()。

由于一个表达式中的未求值操作数没有被求值，所以这个操作数的副作用不被触发。
依赖这些副作用会导致未定义的行为。
不要依赖未求值操作数的副作用。

当需要对象的声明但不需要对象的定义时，未求值表达式操作数会被使用。
比如，在下面的例子中，函数f()被重载，依赖未求值的表达式操作数来选择哪一个函数被重载，进而决定sizeof()表达式的结果。

{% highlight c++ %}
int f(int);
double f(double);
size_t size = sizeof(f(0));
{% endhighlight %}

这样使用不依赖f()的副作用，所以符合这条规范。

# 不符合规范的代码示例 (sizeof)

在这个不符合规范的代码示例中，表达式a++并没有被求值：

{% highlight c++ %}

#include <iostream>
void f() {
  int a = 14;
  int b = sizeof(a++);
  std::cout << a << ", " << b << std::endl;
}
{% endhighlight %}

所以，b初始化后a的值为14。


# 符合规范的解决方案 (sizeof)

在这个符合规范的解决方案中，在sizeof操作符后a被自增了。

{% highlight c++ %}
#include <iostream>
void f() {
  int a = 14;
  int b = sizeof(a);
  ++a;
  std::cout << a << ", " << b << std::endl;
}
{% endhighlight %}


# 不符合规范的代码示例 (decltype)

在这个不符合规范的代码示例中，被decltype修饰的表达式i++并没有被求值。

{% highlight c++ %}

#include <iostream>
 
void f() {
  int i = 0;
  decltype(i++) h = 12;
  std::cout << i;
}
{% endhighlight %}


# 符合规范的解决方案 (decltype)

在这个符合规范的解决方案中，在delctype修饰符之后i被自增，按预期进行了求值。

{% highlight c++ %}
#include <iostream>
 
void f() {
  int i = 0;
  decltype(i) h = 12;
  ++i;
  std::cout << i;
}
{% endhighlight %}


# 例外

**EXP52-CPP-EX1:** 在宏定义或[SFINAE](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-SFINAE)场景下，允许将带有副作用的表达式作为未求值的操作数使用。
尽管这些场景依赖副作用来产生有效代码，但是它们一般不依赖副作用的结果产生的值。

下面的代码是一个在宏定义中使用未求值的操作数的符合规范的例子。

{% highlight c++ %}
void small(int x);
void large(long long x);
  
#define m(x)                                     \
  do {                                           \
    if (sizeof(x) == sizeof(int)) {              \
      small(x);                                  \
    } else if (sizeof(x) == sizeof(long long)) { \
      large(x);                                  \
    }                                            \
  } while (0)
  
void f() {
  int i = 0;
  m(++i);
}
{% endhighlight %}

宏m的展开会导致表达式++i作为sizeof()的一个未求值操作数。
然而，程序员的预期是i只被前向自增一次。
结果是，这是一个安全的宏，并且符合[ PRE31-C. Avoid side effects in arguments to unsafe macros](https://wiki.sei.cmu.edu/confluence/display/c/PRE31-C.+Avoid+side+effects+in+arguments+to+unsafe+macros)。
遵从这条规则对这种例外的代码尤其重要。

下面这个代码在SFINAE情景下使用未求值的操作数来决定一个类型是否可以后缀自增，是符合规范的代码示例。

{% highlight c++ %}
#include <iostream>
#include <type_traits>
#include <utility>
 
template <typename T>
class is_incrementable {
  typedef char one[1];
  typedef char two[2];
  static one &is_incrementable_helper(decltype(std::declval<typename std::remove_cv<T>::type&>()++) *p);
  static two &is_incrementable_helper(...);
   
public:
  static const bool value = sizeof(is_incrementable_helper(nullptr)) == sizeof(one);
};
 
void f() {
  std::cout << std::boolalpha << is_incrementable<int>::value;
}
{% endhighlight %}

在一个is_incrementable实例中，使用后缀自增操作符产生副作用来决定这个类型是否可以进行后缀自增。
然而，这个副作用的结果值被抛弃了，所以副作用只在SFINAE中使用。

# 风险评估

如果一个表达式看起来会产生副作用，但又是一个未求值的操作数，那么结果可能和预期不同。
取决于结果是如何使用的，可能会导致非预期的程序行为。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP52-CPP|低|不太可能|低|P3|L3|


# 相关规则

|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[EXP44-C. Do not rely on side effects in operands to sizeof, _Alignof, or _Generic](https://wiki.sei.cmu.edu/confluence/display/c/EXP44-C.+Do+not+rely+on+side+effects+in+operands+to+sizeof%2C+_Alignof%2C+or+_Generic)|


# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Clause 5, "Expressions" |
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 20.2.5, "Function Template declval" |


# 参考链接

[EXP52-CPP. Do not rely on side effects in unevaluated operands][1]

[Expressions][2]

[C++ 模板特性之 SFINAE][3]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP52-CPP.+Do+not+rely+on+side+effects+in+unevaluated+operands

[2]: https://en.cppreference.com/w/cpp/language/expressions

[3]: http://kaiyuan.me/2018/05/08/sfinae/

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。