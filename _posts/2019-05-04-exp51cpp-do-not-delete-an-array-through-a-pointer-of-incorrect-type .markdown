---
layout: post
title:  "EXP51-CPP 不要使用错误的指针类型来删除一个数组"
date:   2019-05-04 07:10:54 +0800
categories: jekyll update
---

C++标准，[expr.delete], 第3段 [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)], 阐述如下：

> 在第一种替代情况下(删除对象)，如果被删除对象的静态类型和其动态类型不同，
静态类型应当是被删除对象动态类型的基类，并且静态类型应当有虚析构函数，否则行为是未定义的。
在第二种替代情况下(删除数组)，如果被删除对象的动态类型和其静态类型不同，其行为是未定义的。

不要通过和对象动态指针类型不同的静态指针类型来删除一个数组对象。
通过一个不正确类型的指针删除一个数组会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

# 不符合规范的代码示例

在这个不符合规范的代码示例中，一个Derived类型的对象数组被创建，并通过Base\*类型的指针被存储。
虽然Base::~Based()被声明为虚函数，但是它仍然会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。
并且，通过静态类型Base*进行算术运算会违反规则[ CTR56-CPP. Do not use pointer arithmetic on polymorphic objects](https://wiki.sei.cmu.edu/confluence/display/cplusplus/CTR56-CPP.+Do+not+use+pointer+arithmetic+on+polymorphic+objects)。

{% highlight c++ %}

struct Base {
  virtual ~Base() = default;
};
 
struct Derived final : Base {};
 
void f() {
   Base *b = new Derived[10];
   // ...
   delete [] b;
}
{% endhighlight %}



# 符合规范的解决方案

在这个符合规范的解决方案中，b的静态类型是Derived \*，当通过下标访问数组和删除指针时不会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
struct Base {
  virtual ~Base() = default;
};
 
struct Derived final : Base {};
 
void f() {
   Derived *b = new Derived[10];
   // ...
   delete [] b;
}
{% endhighlight %}


# 风险评估

通过不正确的静态类型尝试销毁多态对象数组是未定义的行为。
在实践中，潜在的后果包括非正常的程序执行和内存泄漏。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP51-CPP|低|不太可能|中|P2|L3|


# 相关规则

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[CTR56-CPP. Do not use pointer arithmetic on polymorphic objects](https://wiki.sei.cmu.edu/confluence/display/cplusplus/CTR56-CPP.+Do+not+use+pointer+arithmetic+on+polymorphic+objects)|
|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[OOP52-CPP. Do not delete a polymorphic object without a virtual destructor](https://wiki.sei.cmu.edu/confluence/display/cplusplus/OOP52-CPP.+Do+not+delete+a+polymorphic+object+without+a+virtual+destructor)|


# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 5.3.5, "Delete"|


# 参考链接

[EXP51-CPP. Do not delete an array through a pointer of the incorrect type][1]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP51-CPP.+Do+not+delete+an+array+through+a+pointer+of+the+incorrect+type

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。