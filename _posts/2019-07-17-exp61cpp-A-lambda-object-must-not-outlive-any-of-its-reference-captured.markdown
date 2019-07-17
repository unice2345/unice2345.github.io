---
layout: post
title:  "EXP61-CPP. 一个lambda对象的生命周期不能比其引用捕获的对象的生命周期长"
date:   2019-07-17 06:26:54 +0800
categories: jekyll update
---

Lambda表达式可以在一系列外围作用域（被称作*可达作用域*）范围内捕获到具有自动存储周期的对象，并在lambda函数体中使用。
这些捕获既可以是显式捕获，通过在lambda的*捕获列表*中指定捕获的对象；
也可以是隐式的，通过使用*默认捕获*并在lambda的函数体中引用对象。
当显示或隐式地捕获对象时，默认捕获意味着对象要么通过拷贝(使用=)要么通过引用(使用&)捕获。
当一个对象通过拷贝捕获时，lambda对象会包含一个匿名的非静态数据成员，并被初始化为所捕获对象的值。这个非静态数据成员的生命周期就是lambda对象的生命周期。
然而，当一个对象通过引用被捕获时，引用的生命周期就和lambda对象的生命周期无关了。

因为被捕获的实体是具有自动存储周期的对象（或this）,所以一个一般的准则是返回一个lambda对象的函数（包括通过引用参数返回），或者是将一个lambda对象存储为一个成员变量或全局变量时，不应当通过引用捕获一个实体，因为lambda对象的生命周期经常超出被引用捕获对象的生命周期。

当一个lambda对象的生命周期超出它一个引用捕获对象的生命周期时，一旦该引用捕获的对象被访问，执行该lambda对象的函数调用操作会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。
所以，一个lambda对象不能超出它任意一个引用捕获对象的生命周期。
这是规则[EXP54-CPP. Do not access an object outside of its lifetime](https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP54-CPP.+Do+not+access+an+object+outside+of+its+lifetime)的一个具体实例。

# 不符合规范的代码示例

在这个不符合规范的代码示例中，函数g()返回了一个lambda，该lambda表达式通过引用隐式捕获了自动局部变量i。
当lambda从调用中返回时，它捕获的引用将指向一个生命周期已经结束了的变量，当lambda在f()中被执行时，使用该lambda中的悬挂引用会导致未定义的行为。

{% highlight c++ %}
auto g() {
  int i = 12;
  return [&] {
    i = 100;
    return i;
  };
}
 
void f() {
  int j = g()();
}
{% endhighlight %}


# 符合规范的解决方案

在这个符合规范的解决方案中，lambda没有通过引用捕获i，而是通过拷贝捕获。
结果是，lambda包含了一个隐式的非静态的数据成员，它的生命周期和lambda一致。

{% highlight c++ %}

auto g() {
  int i = 12;
  return [=] () mutable {
    i = 100;
    return i;
  };
}
 
void f() {
  int j = g()();
}
{% endhighlight %}

# 不符合规范的代码示例

在这个不符合规范的代码示例中，一个lambda通过引用捕获了outer lambda中的一个局部变量。
然而，这个inner lambda的生命周期超出了outer lambda及它定义的局部变量的生命周期，导致f()中的inner lambda对象被执行时产生未定义的行为。

{% highlight c++ %}

auto g(int val) {
  auto outer = [val] {
    int i = val;
    auto inner = [&] {
      i += 30;
      return i;
    };
    return inner;
  };
  return outer();
}
 
void f() {
  auto fn = g(12);
  int j = fn();
}
{% endhighlight %}

# 符合规范的解决方案

在这个符合规范的解决方案中，inner lambda通过拷贝捕获i，而不是通过引用捕获。

{% highlight c++ %}

auto g(int val) {
  auto outer = [val] {
    int i = val;
    auto inner = [i] {
      return i + 30;
    };
    return inner;
  };
  return outer();
}
 
void f() {
  auto fn = g(12);
  int j = fn();
}
{% endhighlight %}


# 风险评估

引用一个超出其生命周期的对象会导致攻击者可以执行任意代码。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP61-CPP|高|有可能|高|P6|L2|

# 相关规则

|[CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[EXP54-CPP. Do not access an object outside of its lifetime](https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP54-CPP.+Do+not+access+an+object+outside+of+its+lifetime)|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 3.8, "Object Lifetime"|
||Subclause 5.1.2, "Lambda Expressions"  |

# 参考

[EXP61-CPP. A lambda object must not outlive any of its reference captured objects][1]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP61-CPP.+A+lambda+object+must+not+outlive+any+of+its+reference+captured+objects

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。