---
layout: post
title:  "EXP57-CPP. 不要转换或删除指向非完整类型的指针"
date:   2019-07-02 06:26:54 +0800
categories: jekyll update
---

引用一个不完整类型的对象，也称为*前向声明*，是一种常见的编程实践。
一个常见的使用例子是“pimpl idiom” [[Sutter 00](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Sutter00) ],采用一个不透明的指针用来同公开的API隐藏实现细节。
然而，试图去删除一个指向不完整类型对象的指针会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。
C++标准，[expr.delete],第5段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]阐述如下：

> If the object being deleted has incomplete class type at the point of deletion and the complete class has a non-trivial destructor or a deallocation function, the behavior is undefined.

不要尝试去删除一个指向[不完整类型](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-incompletetype)对象的指针。Although it is well-formed if the class has no nontrivial destructor and no associated deallocation function, it would become undefined behavior were a nontrivial destructor or deallocation function added later. It would be possible to check for a nontrivial destructor at compile time using a static_assert and the std::is_trivially_destructible type trait, but no such type trait exists to test for the presence of a deallocation function.

将指针向下转换成指向不完整类型的指针有类似的警告。
指针向上转换(从派生类转换成基类)是标准的隐式转换操作。
C++允许static_cast实施相反的操作，指针向下转换，根据[expr.static.cast], 第7段。
然而，如果指向的类型是不完整的，编译器无法对类的内存偏移做调整(而这在多重继承存在的时候是必须的), 导致指针无法被有效解引用。

reinterpret_cast一个指针类型在[expr.reinterpret.cast]第7段中被定义为static_cast<cv T *>(static_cast<cv void *>(PtrValue))，意味着reinterpret_cast就是简单的一系列的static_cast操作。
C风格的指向不完整类型对象的指针转换被定义为使用static_cast或者reinterpret_cast （没有指定选择其中哪一个），根据[expr.cast],第5段。

不要试图通过指向不完整类型对象的指针进行转换。
转换操作本身是没问题的，但是解引用转换后的指针可能会导致未定义的行为，如果向下转换没能调整好多重继承。

# 不符合规范的代码示例

在这个不符合规范的代码示例中，一个类试图实现pimpl idiom，但是删除了一个指向不完整类型的指针，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)，如果Body具有一个非平凡的(nontrivial)析构器。

{% highlight c++ %}
class Handle {
  class Body *impl;  // Declaration of a pointer to an incomplete class
public:
  ~Handle() { delete impl; } // Deletion of pointer to an incomplete class
  // ...
};
{% endhighlight %}


# 符合规范的解决方案 (delete)

在这个符合规范的解决方案中，对impl的删除操作被移动到Body被定义的代码部分。

{% highlight c++ %}

class Handle {
  class Body *impl;  // Declaration of a pointer to an incomplete class
public:
  ~Handle();
  // ...
};
 
// Elsewhere
class Body { /* ... */ };
  
Handle::~Handle() {
  delete impl;
}
{% endhighlight %}



# 符合规范的解决方案 (std::shared_ptr)

在这个符合规范的解决方案中，一个std::shared_ptr用来拥有impl的内存。
一个std::shared_ptr可以引用一个不完整类型，但std::unique_ptr不行。

{% highlight c++ %}

#include <memory>
  
class Handle {
  std::shared_ptr<class Body> impl;
  public:
    Handle();
    ~Handle() {}
    // ...
};
{% endhighlight %}


# 不符合规范的代码示例

指针向下转换(从指向基类的指针转换成指向子类的指针)需要按固定大小调整指针的地址，这个固定的大小只能在类继承结构的内存布局已知的情况下才能被决定。
在这个不符合规范的代码示例中，f()接受了一个从get_d()返回的完整类型B的多态指针。
这个指针在被传入g()之前被转换成了不完整类型D。
将指针转换成子类型可能无法成功调整结果指针，当通过调用d->do_something()进行解引用指针时会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).

{% highlight c++ %}
// File1.h
class B {
protected:
  double d;
public:
  B() : d(1.0) {}
};
  
// File2.h
void g(class D *);
class B *get_d(); // Returns a pointer to a D object
 
// File1.cpp
#include "File1.h"
#include "File2.h"
 
void f() {
  B *v = get_d();
  g(reinterpret_cast<class D *>(v));
}
  
// File2.cpp
#include "File2.h"
#include "File1.h"
#include <iostream>
 
class Hah {
protected:
  short s;
public:
  Hah() : s(12) {}
};
 
class D : public Hah, public B {
  float f;
public:
  D() : Hah(), B(), f(1.2f) {}
  void do_something() { std::cout << "f: " << f << ", d: " << d << ", s: " << s << std::endl; }
};
 
void g(D *d) {
  d->do_something();
}
 
B *get_d() {
  return new D;
}
{% endhighlight %}


**实现细节**

当通过[ ClangBB. Definitions#clang](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-clang) 3.8编译并且执行函数f()时，不符合规范的代码示例打印如下内容:

{% highlight shell %}

f: 1.89367e-40, d: 5.27183e-315, s: 0
{% endhighlight %}

类似的，当该例子运行在Microsoft Visual Studio 2015和GCC 6.1.0下时也会打印非预期的值。

# 符合规范的解决方案

这个符合规范的解决方案假设意图是通过使用不完整的类类型来隐藏实现细节。
g()不要求传入D\*,相反要求传入的B*类型。

{% highlight c++ %}
// File1.h -- contents identical.
class B {
protected:
  double d;
public:
  B() : d(1.0) {}
  virtual ~B() {} // 要将B设置为多态，否则对B调用dynamic_cast会出错
};

void f();

// File2.h
void g(class B *); // Accepts a B object, expects a D object
class B *get_d(); // Returns a pointer to a D object

// File1.cpp
#include "File1.h"
#include "File2.h"
 
void f() {
  B *v = get_d();
  g(v);
}
  
// File2.cpp
// ... all contents are identical until ...
void g(B *d) {
  D *t = dynamic_cast<D *>(d);
  if (t) {
    t->do_something();
  } else {
    // Handle error
  }
}
 
B *get_d() {
  return new D;
}

{% endhighlight %}

# 风险评估

转换执行不完整类型的指针或引用会导致损坏的内存地址。
删除一个指向不完整类的指针会导致未定义的行为，如果该类有一个非平凡的析构函数。
这样做会导致程序终止，运行时信号或内存泄漏。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP57-CPP|中|不太可能|中|P4|L3|

# 参考书目

|[Dewhurst 2002](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Dewhurst02)|Gotcha #39, "Casting Incomplete Types"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 4.10, "Pointer Conversions"|
||Subclause 5.2.9, "Static Cast" |
||Subclause 5.2.10, "Reinterpret Cast"|
||Subclause 5.3.5, "Delete"|
||Subclause 5.4, "Explicit Type Conversion (Cast Notation)"|
|[Sutter 2000](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Sutter00)|"Compiler Firewalls and the Pimpl Idiom"|

# 参考

[EXP57-CPP. Do not cast or delete pointers to incomplete classes][1]

[PImpl][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP57-CPP.+Do+not+cast+or+delete+pointers+to+incomplete+classes

[2]: https://en.cppreference.com/w/cpp/language/pimpl

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。