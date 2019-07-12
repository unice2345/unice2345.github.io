---
layout: post
title:  "EXP60-CPP. 不要在跨越执行边界时传递非标准布局类型的对象"
date:   2019-7-11 06:26:54 +0800
categories: jekyll update
---

标准布局类型可以用来和其他编程语言写成的代码进行交流，因为该类型的内存布局是被严格指定的。
C++标准,[class],第7段 [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]，定义标准布局的类如下：

+ 没有虚函数
+ 所有的非静态数据成员具有相同的访问控制级别
+ has no base classes of the same type as the first nonstatic data member,
+ has nonstatic data members declared in only one class within the class hierarchy, and
+ recursively, does not have nonstatic data members of nonstandard-layout type.

*执行边界*是被不同编译器(包括同一个厂商的同一个编译器的不同版本)编译的代码界限。
例如，一个函数被声明在头文件中但是定义在一个运行时加载的库中。
执行边界存在于执行的函数调用点和库中的函数实现处。
这样的边界也被称为[ABI](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-applicationbinaryinterface) (application binary interface)边界，因为它们涉及二进制应用程序的互操作。

不要对非标准布局类型的对象的内存布局做任何假设。
对于对一个编译器编译的而被另外一个编译器引用的对象，这样的假设会引起正确性和移植性问题。
被第一个编译器产生的对象的内存布局，不一定和第二个编译器产生的对象的内存布局相同，即使这两个编译器都[遵循](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-conformingprogram) C++的[实现](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-implementation)。However, some implementations may document binary compatibility guarantees that can be relied on for passing nonstandard-layout objects between execution boundaries.

A special instance of this guidance involves non-C++ code compiled by a different compiler, such as C standard library implementations that are exposed via the C++ standard library. C standard library functions are exposed with C++ signatures, and the type system frequently assists in ensuring that types match appropriately. This process disallows passing a pointer to a C++ object to a function expecting a char * without additional work to suppress the type mismatch. However, some C standard library functions accept a void * for which any C++ pointer type will suffice. Passing a pointer to a nonstandard-layout type in this situation results in indeterminate behavior because it depends on the behavior of the other language as well as on the layout of the given object. For more information, see rule [EXP56-CPP. Do not call a function with a mismatched language linkage](https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP56-CPP.+Do+not+call+a+function+with+a+mismatched+language+linkage).

只有在执行边界的两侧都遵循相同的ABI时才向跨越执行边界传递非标准布局类型的对象。
如果相同版本编译器用来编译执行边界两端，或者编译执行边界两端的编译器跨版本是API兼容的，或者不同的编译器文档说明它们遵循相同的ABI时，允许传递非标准布局类型对象跨越执行边界。

# 不符合规范的代码示例

这个不符合规范的代码示例假设有一个库，它的头文件是library.h, 
还有一个应用程序(application.cpp)，同时库和应用程序不是ABI兼容的。
所以，library.h的内容构成了一个执行边界。
一个非标准布局类型的对象S被跨越执行边界传递。
应用程序创建了这个类型对象的一个实例，然后将对象的引用传递给库中定义的函数，跨越了执行边界。
因为内存布局在跨越边界时不保证是兼容的，会导致未定义的行为。

{% highlight c++ %}
// library.h
struct S {
  virtual void f() { /* ... */ }
};
  
void func(S &s); // Implemented by the library, calls S::f()
  
// application.cpp
#include "library.h"
  
void g() {
  S s;
  func(s);
}
{% endhighlight %}

如果库和应用程序遵从同样的ABI（不管是通过厂商的文档说明，或者使用同样版本的编译器编译库和应用程序），那么这个例子就是兼容的。

# 符合规范的解决方案

因为库和应用程序不遵从同样的ABI，这个符合规范的解决方案修改了库和应用程序，使用标准布局类型。
进一步，它添加了static_assert()来防止未来代码修改时不小心将S修改成非标准布局类型。

{% highlight c++ %}
// library.h
#include <type_traits>
 
struct S {
  void f() { /* ... */ } // No longer virtual
};
static_assert(std::is_standard_layout<S>::value, "S is required to be a standard layout type");
 
void func(S &s); // Implemented by the library, calls S::f()
 
// application.cpp
#include "library.h"
 
void g() {
  S s;
  func(s);
}
{% endhighlight %}

# 不符合规范的代码示例

在这个不符合规范的代码示例中，一个指向非标准布局类型对象的指针被传递给具有"Fortran"语言链接的函数中。
非"C"或"C++"语言链接有时也被实现语义所支持[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]。
如果实现不支持这种语言链接，那么代码就是[病态](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-ill-formed)的。
假设语言链接被支持，任何对传递的对象的操作都可能引起不确定性的行为，可能存在安全问题。

{% highlight c++ %}
struct B {
  int i, j;
};
  
struct D : B {
  float f;
};
  
extern "Fortran" void func(void *);
  
void foo(D *d) {
  func(d);
}
{% endhighlight %}


# 符合规范的解决方案

在这个符合规范的解决方案中，非标准布局类型的对象被序列化到一个局部的标准布局类型对象，然后被传递给Fortran。

{% highlight c++ %}

struct B {
  int i, j;
};
 
struct D : B {
  float f;
};
 
extern "Fortran" void func(void *);
 
void foo(D *d) {
  struct {
    int i, j;
    float f;
  } temp;
  
  temp.i = d->i;
  temp.j = d->j;
  temp.f = d->f;
 
  func(&temp);
}
{% endhighlight %}



# 风险评估

跨越执行边界传递非标准布局类型对象的后果，取决于在被调用者中什么样的操作施加到对象上，也取决于在调用者中后续有什么样的操作施加到对象上。
后果有可能是正确、良性的行为，或者是[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP60-CPP|高|有可能|中|P12|L1|


# 相关规则

|[CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[EXP58-CPP. Pass an object of the correct type to va_start](https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP58-CPP.+Pass+an+object+of+the+correct+type+to+va_start)|


# 参考书目


|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Clause 9, "Classes"|
||Subclause 7.5, "Linkage Specifications" |

# 参考

[EXP60-CPP. Do not pass a nonstandard-layout type object across execution boundaries][1]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP60-CPP.+Do+not+pass+a+nonstandard-layout+type+object+across+execution+boundaries

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。