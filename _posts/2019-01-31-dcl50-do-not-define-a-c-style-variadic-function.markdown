---
layout: post
title: "DCL50-CPP 不要定义C风格的可变参数函数"
date:   2019-01-31 07:00:54 +0800
categories: jekyll update
---

函数可以接受比声明的参数个数更多个的参数，这样的函数称为*可变参数*函数，因为它接收的参数个数是可变化的。
C++提供了两种机制来定义可变参数:函数参数打包和末尾参数使用C风格的省略号。

可变参数函数比较灵活，因为它可以接收可变个数的不同类型的参数。但是，它也是很危险的。
使用C风格省略号的函数(后面称之为*C风格可变参数函数*)没有机制来检查传入参数的类型安全，也没有机制检查传入参数的个数是否符合函数定义的语义。
所以，运行时调用可变参数函数时如果传入了不当的参数，会产生未定义的行为。
这些[未定义行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)可以被[利用](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-exploit)来执行任意代码。

不要定义C风格的可变参数函数。(但是声明C风格的可变参数函数但不给出定义是可以的，因为这样是无害的，在某些未评估的上下文环境下还是有用的。)

在需要给函数传入可变个数参数的场景下，可以使用函数[参数包](https://en.cppreference.com/w/cpp/language/parameter_pack)来解决C风格可变参数函数的问题。
并且，可以使用函数[柯里化](https://zh.wikipedia.org/wiki/%E6%9F%AF%E9%87%8C%E5%8C%96)([function currying](https://en.wikipedia.org/wiki/Currying))方法来替换可变参数函数。
比如，相对于使用C中的printf函数，C++的输出功能实现了只接受单个参数的std::cout::operator<<()操作符。

# 不遵从规范的代码示例

下面这个不遵从规范的代码示例使用了C风格的可变参数实现对一系列int值相加。这个函数一直读取参数，直到遇到0。
在调用这个函数时，在前两个参数后不传入参数0会导致未定义的行为。而且，传入其他非int类型的参数也会导致未定义的行为。

{% highlight c++ %}
#include <cstdarg>
 
int add(int first, int second, ...) {
  int r = first + second; 
  va_list va;
  va_start(va, second);
  while (int v = va_arg(va, int)) {
    r += v;
  }
  va_end(va);
  return r;
}
{% endhighlight %}

# 遵从规范的代码示例 (递归的参数包展开)

在下面的遵从规范的解决方法中，可变参数函数采用函数参数包来实现add()函数，在每个调用点允许实现同样的行为。不像C风格的可变参数函数，在这个解决方法中，即使末尾参数不是0也不会导致未定义的行为。
而且，即使传入的参数不是整数类型，程序会是[ill-formed](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-ill-formed)，在语法上会报错，而不是导致未定义的行为。

{% highlight c++ %}
#include <type_traits>
  
template <typename Arg, typename std::enable_if<std::is_integral<Arg>::value>::type * = nullptr>
int add(Arg f, Arg s) { return f + s; }
  
template <typename Arg, typename... Ts, typename std::enable_if<std::is_integral<Arg>::value>::type * = nullptr>
int add(Arg f, Ts... rest) {
  return f + add(rest...);
}
{% endhighlight %}

这个遵从规范的解决方法利用了[std::enable_if](http://www.cplusplus.com/reference/type_traits/enable_if/)来保证如果传入了非整数的参数，程序会出现语法错误。

# 遵从规范的代码示例 (花括号环绕初始化器的参数包展开)

另外一种遵从规范的解决方法并不需要递归展开参数包，而是将参数包展开成一系列值，作为花括号初始化器列表的一部分。由于花括号初始化器列表不允许向窄转换，所以类型安全得以保证，尽管std::enable_if没有应用到任意一个可变参数上。

{% highlight c++ %}
#include <type_traits>

template<typename Arg, typename... Ts, typename std::enable_if<std::is_integral<Arg>::value>::type * = nullptr>
int add(Arg i, Arg j, Ts... all) {
    int values[] = {j, all...};
    int r = i;
    for (auto v: values) {
        r += v;
    }
    return r;
}
{% endhighlight %}

# 例外

**DCL50-CPP-EX1:** 如果函数有一个C语言的外部链接，那么该函数定义成C风格的可变参数是允许的。
比如，该函数被用在C库API中，用C++语言来实现。

**DCL50-CPP-EX2:** 正如规范正文中所阐述的，C风格可变参数的函数被声明却未实现是允许的。比如，
当一个函数调用出现在未评估的上下文中（例如作为sizeof表达式的参数），[重载决议](https://en.cppreference.com/w/cpp/language/overload_resolution)被执行来决定调用的结果类型，但是这时并不需要函数的定义。有些利用[SFINAE](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-SFINAE)的模板元编程技巧采用可变函数声明来实现编译期间的类型查询，如下面例子所示。

{% highlight c++ %}
template <typename Ty>
class has_foo_function {
  typedef char yes[1];
  typedef char no[2];
 
  template <typename Inner>
  static yes& test(Inner *I, decltype(I->foo()) * = nullptr); // Function is never defined.
 
  template <typename>
  static no& test(...); // Function is never defined.
 
public:
  static const bool value = sizeof(test<Ty>(nullptr)) == sizeof(yes);
};
{% endhighlight %}

在这个例子中，value的值取决于哪个test()会被重载。Inner \*I声明允许使用decltype修饰参数I，
产生指向某种类型（可以是void类型，默认为nulltype）的指针。但是，如果没有声明Inner::foo(),
decltype修饰符就是非法的，根据SFINAE规则，这个test()就不会成为候选的重载函数。这样的结果是，
C风格可变参数的test()成为重载候选集中的唯一函数。这两个test()函数都只是被声明而没有定义，因为
在这种未评估上下文环境中使用不需要进行定义。

# 风险评估

不正确的使用可变参数函数可能导致[程序异常终止](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-abnormaltermination), 非有意的信息泄露和执行任意代码。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL50-CPP|高|一般可能|中|P12|L1|

**自动检测**

略

**相关漏洞**

略

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)| 5.2.2节 "函数调用" |
|| 14.5.3节 "变参模板" |


# 参考链接

[DCL50-CPP. Do not define a C-style variadic function][1]

[SFINAE][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL50-CPP.+Do+not+define+a+C-style+variadic+function
[2]: https://en.cppreference.com/w/cpp/language/sfinae

