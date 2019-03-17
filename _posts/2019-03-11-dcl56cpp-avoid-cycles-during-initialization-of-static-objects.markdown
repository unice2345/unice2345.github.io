---
layout: post
title:  "DCL56-CPP 初始化静态对象时要避免循环"
date:   2019-03-11 06:26:54 +0800
categories: jekyll update
---

C++标准, [stmt.dlc], 第4段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]阐述如下：

> 具有静态存储期(3.7.1)或线程存储期(3.7.2)的块级作用域的变量的零值初始化(zero-initialization(8.5))先于其他的初始化执行。
对于具有静态存储期的块作用域的实体的常量初始化(constant initialization (3.6.2)), 如果可以执行的话，会在进入块区域之前先执行。
对于其他具有静态或线程存储期的块作用域变量，它们的早期初始化可以由实现来决定；
同样条件下，具有静态或线程存储期的空间作用域(3.6.2)变量，静态初始化可以由实现来决定。
除此之外，这样的变量在第一次控制经过声明时(???)被初始化, 这样的变量被认为在整个初始化完成时被初始化的。如果初始化时抛出了异常，那么初始化是不完整的，
在下次控制进入到声明的时刻时会再次尝试初始化。
如果在初始化变量时，控制是并发进入到声明处的，并发执行应该等待初始化结束。
如果在变量初始化时，控制是递归地进入声明处的，行为是未定义的。

在静态变量初始化时不要重入一个函数。如果在一个函数内部的静态对象进行常量初始化时重入该函数，
那么程序的行为是[未定义的](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).
触发未定义的行为并不需要无限递归，在初始化部分完成时函数只需要重入一次就可以触发未定义行为。
由于变量的线程安全初始化特性，一次递归的调用常会由于锁定了非递归的线程同步原子量而导致[死锁](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-deadlock).

另外，C++标准，[basic.start.init]，第2段部分阐述如下：

> 动态初始化一个具有静态存储期的非局部变量可以是有序或无序的。
显示特化的类模板的静态成员变量是有序初始化的。
其他类模板的静态成员变量(比如，隐式或显示实例化的特化implicitly or explicitly instantiated specializations)是无序初始化的。
其他具有静态存储期的非局部变量是有序初始化的。
定义在一个翻译单元中的有序初始化的变量，应该按照他们在翻译单元中的定义顺序进行初始化。
如果一个程序启动了一个线程，那么一个变量的后续初始化相对于定义在其他翻译单元中的变量的初始化是无序的。
其他情况下，定义在不同翻译单元中的变量初始化是以非确定的顺序进行的。
如果一个程序启动了一个线程，一个变量的后续的无序初始化是和其他动态初始化是非顺序进行的。
除此以外，一个变量的无序初始化和其他动态初始化是以非确定性的顺序进行的。

对于具有动态初始化的静态对象，不要使它们的初始化相互依赖，除非它们是有序的初始化。
无序的初始化，尤其是普遍存在于翻译单元边界的，会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-unspecifiedbehavior)。

# 不遵从规范的示例代码

下面这个例子展示了尝试用缓存实现高效factorial函数的方法。因为初始化静态本地数组cahe时涉及到了递归，这个函数的行为是未定义的，尽管递归不是无限的。

{% highlight c++ %}
#include <stdexcept>
  
int fact(int i) noexcept(false) {
  if (i < 0) {
    // Negative factorials are undefined.
    throw std::domain_error("i must be >= 0");
  }
  
  static const int cache[] = {
    fact(0), fact(1), fact(2), fact(3), fact(4), fact(5),
    fact(6), fact(7), fact(8), fact(9), fact(10), fact(11),
    fact(12), fact(13), fact(14), fact(15), fact(16)
  };
  
  if (i < (sizeof(cache) / sizeof(int))) {
    return cache[i];
  }
  
  return i > 0 ? i * fact(i - 1) : 1;
}
{% endhighlight %}

在[Microsoft Visual Studio](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-msvc) 2015和[GCC](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-gcc) 6.1.0上，当以线程安全的方式初始化静态变量时发生递归初始化cache导致死锁。

# 遵从规范的示例代码

这个遵从规范的解决方案避免了初始化静态本地数组cache，它先利用零值初始化，然后通过判零来决定每个成员是否已经被赋值，如果没有赋值，再递归的计算它的值。这样，在有缓存值时就返回缓存值，否则就按需要计算它的值。

{% highlight c++ %}
#include <stdexcept>
  
int fact(int i) noexcept(false) {
   if (i < 0) {
    // Negative factorials are undefined.
    throw std::domain_error("i must be >= 0");
  }
 
  // Use the lazy-initialized cache.
  static int cache[17];
  if (i < (sizeof(cache) / sizeof(int))) {
    if (0 == cache[i]) {
      cache[i] = i > 0 ? i * fact(i - 1) : 1;
    }
    return cache[i];
  }
  
  return i > 0 ? i * fact(i - 1) : 1;
}
{% endhighlight %}

# 不遵从规范的示例代码

在这个不遵从规范的示例代码中，file1.cpp中的numWheels的值依赖于c的初始化。
然而，因为c是被定义在和numWheels不同的翻译单元中(file2.cpp)，
并不能保证numWheels被初始化(调用c.get_num_wheels())之前，c已经被初始化(通过调用get_default_car())。
这种现象被称作"[静态初始化顺序问题(static initialization order fiasco)](https://isocpp.org/wiki/faq/ctors#static-init-order)",会导致未定义的行为。

{% highlight c++ %}
// file.h
#ifndef FILE_H
#define FILE_H
  
class Car {
  int numWheels;
  
public:
  Car() : numWheels(4) {}
  explicit Car(int numWheels) : numWheels(numWheels) {}
  
  int get_num_wheels() const { return numWheels; }
};
#endif // FILE_H
  
// file1.cpp
#include "file.h"
#include <iostream>
  
extern Car c;
int numWheels = c.get_num_wheels();
  
int main() {
  std::cout << numWheels << std::endl;
}
  
// file2.cpp
#include "file.h"
  
Car get_default_car() { return Car(6); }
Car c = get_default_car();
{% endhighlight %}

**实现细节**

输出到标准输出流的值取决于翻译单元被链接的顺序。
比如，在x86 Linux上使用[Clang](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-clang) 3.8.0，命令clang++ file1.cpp file2.cpp && ./a.out会打印0，而命令clang++ file2.cpp file1.cpp && ./a.out会打印6.

# 遵从规范的解决方案

这个遵从规范的解决方案使用了"首次使用时构建"的习惯用法来解决静态初始化的顺序问题。
file.h和file2.cpp中的代码不变，只是将file1.cpp中的静态numWheels移动到一个函数体内部。
这样，当控制流到达声明点时，numWheels的初始化已经确保发生了，保证了控制的顺序。
全局对象c在main()执行之前初始化，所以当get_num_wheels()被调用时，c确保已经被动态初始化了。

{% highlight c++ %}
// file.h
#ifndef FILE_H
#define FILE_H
 
class Car {
  int numWheels;
 
public:
  Car() : numWheels(4) {}
  explicit Car(int numWheels) : numWheels(numWheels) {}
 
  int get_num_wheels() const { return numWheels; }
};
#endif // FILE_H
 
// file1.cpp
#include "file.h"
#include <iostream>
 
int &get_num_wheels() {
  extern Car c;
  static int numWheels = c.get_num_wheels();
  return numWheels;
}
 
int main() {
  std::cout << get_num_wheels() << std::endl;
}
 
// file2.cpp
#include "file.h"
 
Car get_default_car() { return Car(6); }
Car c = get_default_car();
{% endhighlight %}


# 风险评估

当一个静态对象在初始化时递归的重入函数会导致攻击者可以造成程序崩溃或[拒绝服务](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-denial-of-service).
不确定顺序的动态初始化能够导致未定义的行为，因为访问了未初始化的对象。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL56-CPP|低|不太可能|中|P2|L3|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[CERT Oracle Coding Standard for Java](https://www.securecoding.cert.org/confluence/display/java/SEI+CERT+Oracle+Coding+Standard+for+Java)|[DCL00-J. Prevent class initialization cycles](DCL00-J. Prevent class initialization cycles)|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 3.6.2, "Initialization of Non-local Variables"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 6.7, "Declaration Statement"|


# 参考链接：


[DCL56-CPP. Avoid cycles during initialization of static objects] [1]

[constant initialization] [2]

[zero initialization] [3]

[Non-local variables] [4]

[Initialization: Dynamic initialization, Early dynamic initialization, Deferred dynamic initialization] [5]

[Construct On First Use] [6]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL56-CPP.+Avoid+cycles+during+initialization+of+static+objects

[2]: https://en.cppreference.com/w/cpp/language/constant_initialization

[3]: https://en.cppreference.com/w/cpp/language/zero_initialization

[4]: https://en.cppreference.com/w/cpp/language/initialization#Non-local_variables

[5]: https://en.cppreference.com/w/cpp/language/initialization

[6]: https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Construct_On_First_Use