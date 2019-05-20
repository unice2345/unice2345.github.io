---
layout: post
title:  "EXP54-CPP 不要访问生命周期之外的对象"
date:   2019-05-14 07:10:54 +0800
categories: jekyll update
---

每个对象都有一个生命周期，在生命周期内对象可以以定义良好的方式使用。
对象的生命周期始于足够的适当对齐的内存被分配给对象，并且初始化已完成。
当对象的析构函数（如果有的话）被调用并且对象的内存被释放或被重新使用了，对象的生命周期结束。
在生命周期之外使用对象或指向对象的指针，会经常导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

C++标准，[basic.life], 第5段 [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)], 阐述了指针的生命周期规则:

> 在对象的生命周期开始之前但其将要占据的存储已经分配之后，或者在对象的生命周期已经结束之后但其所曾占据的存储被重用或释放之前，任何指向对象将要或曾占据的存储的指针，只能以有限的方式进行使用。 
对于正在被构建或析构的对象，参见12.7节。
除此之外，一个指向已分配内存的指针，或者将指针作为void* 类型使用时是定义良好的。
对这样的指针进行间接使用(Indirection？？？)是允许的，但是结果的lvalue只能以有限的方式使用，如下所述。
在以下几种情况下，程序具有未定义的行为：
  - 对象将成为或者曾经是具有非平凡析构函数的类，并且指针被用作*delete表达式*的操作数,
  - 指针被用来访问一个对象的非静态数据成员或者调用一个对象的非静态成员函数，或者，
  - 指针被隐式转换为一个指向虚基类的指针，或者，
  - 指针被用作static_cast的操作数，除了转换成cv void类型的指针，或者转换成cv void指针后又接着转换成cv char或cv unsigned char类型的指针，或者，
  - 指针被用作dynamic_cast的操作数。

第6段描述了非指针的生命周期规则：

> 类似的，在对象的生命周期开始之前但其将要占据的存储已经分配之后，或者在对象的生命周期已经结束之后但其所曾占据的存储被重用或释放之前，任何指向初始对象的泛左值(glvalue)只能以有限的几种方式使用。
对于正在被构建或析构的对象，参见12.7节。
除此之外，指向已分配内存的一个泛左值，并且使用不依赖其值的泛左值的属性是定义良好的。
在以下几种情况下，程序具有未定义的行为：
  - 在这样的泛左值上施加左值向右值转换,
  - 泛左值被用来访问一个对象的非静态数据成员或者调用一个对象的非静态成员函数，或者，
  - 泛左值被绑定到一个虚基类函数的引用上，或者，
  - 泛左值被用作dynamic_cast或者typeid的操作数。

不用在对象的生命周期之外使用对象，除了以上面所述的定义良好的方式使用。


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个指向对象的指针在其生命周期之前用来调用对象的非静态成员函数，
导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}

struct S {
  void mem_fn();
};
  
void f() {
  S *s;
  s->mem_fn();
}
{% endhighlight %}


# 符合规范的解决方案 

在这个符合规范的解决方案中，在调用S::mem_fn()之前先给指针分配了内存。

{% highlight c++ %}
struct S {
  void mem_fn();
};
  
void f() {
  S *s = new S;
  s->mem_fn();
  delete s;
}
{% endhighlight %}


一个改进的解决方案是不直接动态分配内存，而是使用一个自动局部变量来获得内存并进行初始化。
如果需要一个指针，可以采用智能指针，比如std::unique_ptr，也是一种很好的改进。
然而这些改进的解决方案不会展示出生命周期的使用。


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个指向对象的指针在对象的生命周期结束之后，被转换成了一个指向虚基类的指针，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
struct B {};
  
struct D1 : virtual B {};
struct D2 : virtual B {};
  
struct S : D1, D2 {};
  
void f(const B *b) {}
  
void g() {
  S *s = new S;
  // Use s
  delete s;
  
  f(s);
}
{% endhighlight %}

尽管事实上f()没有使用对象，但是指针作为参数传入f()已经足够触发非定义的行为了。

# 符合规范的解决方案 

在这个符合规范的解决方案中，s的生命周期被扩展了以覆盖f()的调用。

{% highlight c++ %}
struct B {};
  
struct D1 : virtual B {};
struct D2 : virtual B {};
  
struct S : D1, D2 {};
  
void f(const B *b) {}
  
void g() {
  S *s = new S;
  // Use s
  f(s);
  
  delete s;
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个局部变量的地址从f()中返回。
当这个结果指针被传递到h()中时，左值向右值转换发生到i上，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
int *g() {
  int i = 12;
  return &i;
}
  
void h(int *i);
  
void f() {
  int *i = g();
  h(i);
}
{% endhighlight %}


当一个指向具有自动存储期的对象的指针从函数返回时（如本例所示），一些编译器会产生诊断信息。

# 符合规范的解决方案 

在这个符合规范的解决方案中，从g()返回的自动变量具有静态存储期而不是自动存储期，
扩展了它的生命周期，足够在f()中使用。

{% highlight c++ %}

int *g() {
  static int i = 12;
  return &i;
}
  
void h(int *i);
  
void f() {
  int *i = g();
  h(i);
}
{% endhighlight %}



# 不符合规范的代码示例 

一个从初始化列表构建的std::initializer_list<>对象，它的实现相当于是分配了一个临时数组，
并把它传递给std::initializer_list<>构造器。
这个临时数组和其他的临时对象具有相同的生命周期，除了那些从扩展了生命周期（比如给临时对象绑定了引用[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]）的数组初始化的std::initializer_list<>对象。

在这个不符合规范的代码示例中，一个std::initializer_list<int>类型的成员变量是在构造器的构造初始化列表(ctor-initializer)中通过列表初始化的。
在这些条件下，一旦构造器退出，临时数组的生命周期也就结束了，
所以访问std::initializer_list<int>成员变量的任何一个元素都会导致未定义的行为]。


{% highlight c++ %}

#include <initializer_list>
#include <iostream>
 
class C {
  std::initializer_list<int> l;
   
public:
  C() : l{1, 2, 3} {}
   
  int first() const { return *l.begin(); }
};
 
void f() {
  C c;
  std::cout << c.first();
}
{% endhighlight %}


# 符合规范的解决方案 

在这个符合规范的解决方案中，std::initializer_list\<int\>成员变量被std::vector\<int\>替代，
std::vector\<int\>将初始化列表中的元素拷贝到容器内，而不是依赖临时数组，那样会引起悬挂引用。

{% highlight c++ %}
#include <iostream>
#include <vector>
  
class C {
  std::vector<int> l;
   
public:
  C() : l{1, 2, 3} {}
   
  int first() const { return *l.begin(); }
};
  
void f() {
  C c;
  std::cout << c.first();
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个lambda对象被存储到一个函数对象中，
这个函数对象随后被调用(执行lambda)来获取一个指向一个值的常量引用。
Lambda对象返回一个int值，这个值被存储到一个临时的int对象中，
并被绑定到由函数对象指定的const int&的返回类型上。
然而，临时对象的生命周期并没有扩展到超过函数对象调用的返回之后，
这样当访问返回值时会导致未定义的行为。

{% highlight c++ %}
#include <functional>
  
void f() {
  auto l = [](const int &j) { return j; };
  std::function<const int&(const int &)> fn(l);
  
  int i = 42;
  int j = fn(i);
}
{% endhighlight %}


# 符合规范的解决方案 

在这个符合规范的解决方案中，std::function对象返回了一个int而不是const int&，
这样保证了这个值被拷贝而不是绑定到一个临时的引用上。
一个替代的解决方案是直接调用lambda而不是通过std::function<>对象。

{% highlight c++ %}

#include <functional>
  
void f() {
  auto l = [](const int &j) { return j; };
  std::function<int(const int &)> fn(l);
  
  int i = 42;
  int j = fn(i);
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，自动变量s的构造函数没有被调用，
这是由于goto语句的作用导致调用过程没有到达这个临时变量的声明处。
因为构造函数没有被调用，所以s的生命周期没有开始。
所以，调用S::f()使用了生命周期之外的对象，会导致未定义的行为。

{% highlight c++ %}
class S {
  int v;
  
public:
  S() : v(12) {} // Non-trivial constructor
  
  void f();
};  
  
void f() {
  
  // ...  
  
  goto bad_idea;  
  
  // ...
  
  S s; // Control passes over the declaration, so initialization does not take place.  
  
  bad_idea:
    s.f();
}
{% endhighlight %}


# 符合规范的解决方案 

这个符合规范的解决方案保证了执行本地跳转之前s已经被合适地初始化了。

{% highlight c++ %}
class S {
  int v;
  
public:
  S() : v(12) {} // Non-trivial constructor
   
  void f();
};  
  
void f() {
  S s;
  
  // ...
  
  goto bad_idea;
  
  // ...
  
  bad_idea:
    s.f();
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，f()被调用处理一定范围的类型S的对象。
这些对象使用std::copy被拷贝到一个临时的缓冲区中，当处理完这些对象后，临时缓冲区被释放。
然而，通过std::get_temporary_buffer()返回的缓冲区内不包含已初始化了的S类型对象，
所以当std::copy解引用目的迭代器时会导致未定义的行为，因为被目的迭代器解引用的对象还没有开始生命周期。这是因为给对象分配空间时，没有构造器或初始化器被调用。

{% highlight c++ %}

#include <algorithm>
#include <cstddef>
#include <memory>
#include <type_traits>
  
class S {
  int i;
 
public:
  S() : i(0) {}
  S(int i) : i(i) {}
  S(const S&) = default;
  S& operator=(const S&) = default;
};
 
template <typename Iter>
void f(Iter i, Iter e) {
  static_assert(std::is_same<typename std::iterator_traits<Iter>::value_type, S>::value,
                "Expecting iterators over type S");
  ptrdiff_t count = std::distance(i, e);
  if (!count) {
    return;
  }

  // Get some temporary memory.
  auto p = std::get_temporary_buffer<S>(count);
  if (p.second < count) {
    // Handle error; memory wasn't allocated, or insufficient memory was allocated.
    return;
  }
  S *vals = p.first;
   
  // Copy the values into the memory.
  std::copy(i, e, vals);
   
  // ...
   
  // Return the temporary memory.
  std::return_temporary_buffer(vals);
}
{% endhighlight %}


**实现细节**

std::get_temporary_buffer()和std::copy()的一个合理的实现代码跟下面这个例子的行为类似（不包含错误检查）:


```

unsigned char *buffer = new (std::nothrow) unsigned char[sizeof(S) * object_count];
S *result = reinterpret_cast<S *>(buffer);
while (i != e) {
  *result = *i; // Undefined behavior
  ++result;
  ++i;
}
```

解引用result是未定义的行为，因为内存指向的不是一个在生命周期内的S类型对象。


# 符合规范的解决方案 (std::uninitialized_copy())


在这个符合规范的解决方案中，std::uninitialized_copy()被用来执行拷贝，而不是用std::copy()。
这样做保证了对象通过占位new进行初始化而不是解引用未初始化的内存。
简单起见，移除了和上面不符合规范代码示例中相同的代码。

{% highlight c++ %}

//...
  // Copy the values into the memory.
  std::uninitialized_copy(i, e, vals);
// ...
{% endhighlight %}


# 符合规范的解决方案 (std::raw_storage_iterator)

这个符合规范的解决方案使用std::copy()同时将std::raw_storage_iterator作为目的迭代器，
实现了和使用std::uninitialized_copy()同样的定义良好的结果。
和上面的例子一样，简单起见，移除了和上面不符合规范代码示例中相同的代码。

{% highlight c++ %}
//...
  // Copy the values into the memory.
  std::copy(i, e, std::raw_storage_iterator<S*, S>(vals));
// ...
{% endhighlight %}


# 风险评估

引用一个超出其生命周期的对象可能导致攻击者执行任意代码。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP54-CPP|高|有可能|高|P6|L2|


# 相关规则

|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[DCL30-C. Declare objects with appropriate storage durations](https://wiki.sei.cmu.edu/confluence/display/c/DCL30-C.+Declare+objects+with+appropriate+storage+durations)|


# 参考书目

|[Coverity 2007]||
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 3.8, "Object Lifetime"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 8.5.4, "List-Initialization" |


# 参考链接

[EXP54-CPP. Do not access an object outside of its lifetime][1]

[生存期][2]

[lifetime][3]

[Value categories][4]

[std::initializer_list][5]

[std::uninitialized_copy][6]

[std::raw_storage_iterator][7]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP54-CPP.+Do+not+access+an+object+outside+of+its+lifetime

[2]: https://zh.cppreference.com/w/cpp/language/lifetime

[3]: https://en.cppreference.com/w/cpp/language/lifetime

[4]: https://en.cppreference.com/w/cpp/language/value_category

[5]: https://en.cppreference.com/w/cpp/utility/initializer_list

[6]: https://en.cppreference.com/w/cpp/memory/uninitialized_copy

[7]: https://en.cppreference.com/w/cpp/memory/raw_storage_iterator

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。