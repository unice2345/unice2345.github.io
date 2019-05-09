---
layout: post
title:  "EXP53-CPP 不要读取未初始化的内存"
date:   2019-05-09 06:10:54 +0800
categories: jekyll update
---

局部的自动变量如果在被初始化之前读取了，被认为是具有非预期的值。
C++标准，[dcl.init], 第12段 [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)], 阐述如下：

> 如果一个对象没有指定的初始化器，这个对象被默认初始化。
如果一个对象具有自动或动态的存储期，这个对象具有*不确定的值*，如果这个对象没有被实施初始化，那么这个对象会一直保持不确定的值，直到这个值被替代。
如果一个不确定的值是通过求值产生的，其行为是未定义的，以下情形除外：
- 如果一个无符号窄字符类型的不确定的值是通过以下求值产生：
  - 一个条件表达式的第二个或第三个操作数，
  - 一个逗号表达式的右操作数，
  - 向无符号窄字符类型进行cast或转换的操作数,
  - 一个弃值表达式

> 那么操作的结果是一个不确定的值。
- 如果一个无符号窄字符类型的不确定值是由一个简单赋值操作符的右操作数求值产生的，
并且这个赋值操作符的第一个操作数是一个无符号窄字符类型的左值(lvalue)，一个不确定的值替代被左操作数指向的对象的值。
- 如果一个无符号窄字符类型的不确定值是由初始化一个窄字符类型的对象时的初始化表达式求值产生的，
这个对象被初始化为一个不确定的值。

默认初始化一个对象在第7段的同一个子小节描述：

> *默认初始化*一个T类型的对象意味着：
- 如果T是一个(也许是cv修饰的)类, T的默认构造函数被调用(如果T没有默认构造函数，或者重载决议导致歧义或一个被删除或不可访问的函数，那么初始化是病态的);
- 如果T是一个数组类型，其中的每个元素被默认初始化；
- 其他情形下，初始化不执行。

> 如果程序调用了一个const修饰的类型T对象的默认初始化，T应该是一种有用户提供的默认构造函数的类。

所以，具有自动或动态存储期的类型T的对象，在它的值被作为表达式的一部分进行读取之前必须被显示初始化，除非T是类、类的数组或者是一个窄字符类型。
如果T是一个窄字符类型，它可以用来初始化一个窄字符类型的对象，导致两个对象都具有[不确定的值](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-indeterminatevalue)。
这个技术可以用来在不触发[未定义行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)的前提下实现像std::memcpy()一样的拷贝构造函数。

另外，当*new-initialized*被忽略时（即不使用初始化器时），通过new表达式动态分配的内存是默认初始化的。
通过标准库函数std::calloc()分配的内存被零初始化。
通过标准库函数std::realloc()分配的内存保存了原指针的值但是可能并未初始化全部范围的内存。
通过其他方式分配的内存(std::malloc(), 分配器对象，new操作符等)被认为是默认初始化的。

静态对象或具有线程存储期的对象在其他初始化发生之前被零初始化[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)]，在它们的值被读取之前不需要进行显示初始化。

为创造熵(???)而读取未初始化的变量是有问题的，因为这些内存访问可能由于编译器优化而被删除。
VU925211就是一个由这个编码错误造成的[漏洞](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-vulnerability)例子 [[VU#925211](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-VU925211)]。


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个未初始化的局部变量作为打印值表达式的一部分被求值了，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}

#include <iostream>
  
void f() {
  int i;
  std::cout << i;
}
{% endhighlight %}

# 符合规范的解决方案 

在这个符合规范的解决方案中，对象的值被打印之前先进行了初始化。

{% highlight c++ %}

#include <iostream>
  
void f() {
  int i = 0;
  std::cout << i;
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个int\*对象通过*new表达式*分配，但是它指向的内存并未被初始化。
对象的指针值和指针所指向的值被打印到标准输出流。
打印指针值是定义良好的，但是试图打印指针指向的值会产生一个[不确定的值](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-indeterminatevalue)，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}

#include <iostream>
  
void f() {
  int *i = new int;
  std::cout << i << ", " << *i;
}

{% endhighlight %}


# 符合规范的解决方案 

在这个符合规范的解决方案中，内存被打印之前被直接初始化为值12。

{% highlight c++ %}
#include <iostream>
  
void f() {
  int *i = new int(12);
  std::cout << i << ", " << *i;
}

{% endhighlight %}

初始化一个由*new表达式*生成的对象可以通过在分配类型后面加圆括号(可以是空的)或花括号实现。
这会导致被指向的对象进行直接初始化，如果初始化忽略了传入值那么会对对象进行零初始化，就像下面的代码所示：

```
int *i = new int(); // zero-initializes *i
int *j = new int{}; // zero-initializes *j
int *k = new int(12); // initializes *k to 12
int *l = new int{12}; // initializes *l to 12
```


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，类成员变量c没有通过在默认构造函数中的*构造初始化器*进行显示初始化。
尽管局部变量s被默认初始化了，但在调用S::f()中使用c会导致对具有不确定值的对象进行求值，导致未定义的行为。

{% highlight c++ %}

class S {
  int c;
  
public:
  int f(int i) const { return i + c; }
};
  
void f() {
  S s;
  int i = s.f(10);
}
{% endhighlight %}


# 符合规范的解决方案 

在这个符合规范的解决方案中，S有一个默认构造函数，会初始化成员变量c。

{% highlight c++ %}

class S {
  int c;
  
public:
  S() : c(0) {}
  int f(int i) const { return i + c; }
};
  
void f() {
  S s;
  int i = s.f(10);
}
{% endhighlight %}


# 风险评估

读取未初始化的变量是未定义的行为并可能导致非预期的程序行为。
在一些情况下，这些[安全缺陷](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-securityflaw)可能会允许执行任意代码。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP53-CPP|高|有可能|中|P12|L1|


# 相关规则

|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[EXP33-C. Do not read uninitialized memory](https://wiki.sei.cmu.edu/confluence/display/c/EXP33-C.+Do+not+read+uninitialized+memory)|


# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Clause 5, "Expressions" |
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 5.3.4, "New"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 8.5, "Initializers"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 12.6.2, "Initializing Bases and Members" |
|[[Lockheed Martin 2005](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-LockheedMartin05)]|Rule 142, All variables shall be initialized before use|


# 参考链接

[EXP53-CPP. Do not read uninitialized memory][1]

[default initialization][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP53-CPP.+Do+not+read+uninitialized+memory

[2]: https://en.cppreference.com/w/cpp/language/default_initialization

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。