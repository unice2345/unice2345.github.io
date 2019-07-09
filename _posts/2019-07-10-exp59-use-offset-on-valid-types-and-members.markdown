---
layout: post
title:  "EXP59-CPP 在正确的类型和成员上使用offsetof()"
date:   2019-07-10 06:26:54 +0800
categories: jekyll update
---

offset宏被C标准定义为一种可移植的方式来确定一个对象的起始到该对象的一个成员的位置偏移，以bytes为单位。
C标准,第7.17节，第3段 [[ISO/IEC 9899:1999](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC9899-1999)] 部分阐述如下：

> offsetof(type, member-designator) which expands to an integer constant expression that has type size_t, the value of which is the offset in bytes, to the structure member (designated by member-designator), from the beginning of its structure (designated by type). The type and member designator shall be such that given static type t; then the expression &(t.member-designator) evaluates to an address constant. (If the specified member is a bit-field, the behavior is undefined.)

C++标准, [support.types],第4段,[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]在C标准的基础上又添加了额外的限制:

> The macro offsetof(type, member-designator) accepts a restricted set of type arguments in this International Standard. If type is not a standard-layout class, the results are undefined. The expression offsetof(type, member-designator) is never type-dependent and it is value-dependent if and only if type is dependent. The result of applying the offsetof macro to a field that is a static data member or a function member is undefined. No operation invoked by the offsetof macro shall throw an exception and noexcept(offsetof(type, member-designator)) shall be true.

当指定offsetof()宏的type参数时，只应传入标准布局类。
标准布局类的完整描述可以在C++标准的[class]字句的第7段找到，
或者也可以通过std::is_stardard_layout<>类型trait来判断类型是否为标准布局类。
当指定offsetof()宏的成员标识符参数时，不要传入位域、静态数据成员或函数成员。向offsetof()宏传入无效的类型或成员是[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).

# 不符合规范的代码示例

在这个不符合规范的代码示例中，传入offsetof()宏的类型不是标准布局类，导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).

{% highlight c++ %}
#include <cstddef>
  
struct D {
  virtual void f() {}
  int i;
};
  
void f() {
  size_t off = offsetof(D, i);
  // ...
}
{% endhighlight %}

**实现细节**

这个不符合规范的代码示例在x86上使用Microsoft Visual Studio 2015编译并打开/Wall参数时不发出任何警告，由于类型D的虚函数表的存在，导致off值为4.

# 符合规范的解决方案

在D中决定到i的偏移量是不可能的，因为D不是一个标准布局类。
然而，如果这个功能对应用来说是紧要的，可以在D内部创造一个标准布局类，
如下面这个符合规范的解决方案所示：

{% highlight c++ %}

#include <cstddef>
 
struct D {
  virtual void f() {}
  struct InnerStandardLayout {
    int i;
  } inner;
};
 
void f() {
  size_t off = offsetof(D::InnerStandardLayout, i);
  // ...
}
{% endhighlight %}


# 不符合规范的代码示例

在这个不符合规范的代码示例中，到i的偏离量被计算以便使一个值可以存储到buffer中的这个偏移位置处。
然而，由于i是类的一个静态数据成员，这个例子会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior).
根据C++标准，[class.static.data]，第1段[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)],静态数据成员不是一个类的子对象的一部分。

{% highlight c++ %}

#include <cstddef>
  
struct S {
  static int i;
  // ...
};
int S::i = 0;
  
extern void store_in_some_buffer(void *buffer, size_t offset, int val);
extern void *buffer;
  
void f() {
  size_t off = offsetof(S, i);
  store_in_some_buffer(buffer, off, 42);
}

{% endhighlight %}

**实现细节**

这个不符合规范的代码示例，在x86下使用 Microsoft Visual Studio 2015 编译并打开/Wall开关不会产生告警，导致off是一个很大的值，代表null指针地址0和静态变量S::i地址之间的偏移量。

# 符合规范的解决方案

由于静态数据成员不是类内存布局的一部分，而是一个单独实体，
所以这个符合规范的解决方案传入静态成员变量的地址作为要存储数据的buffer，并给offset传入0.

{% highlight c++ %}

#include <cstddef>
  
struct S {
  static int i;
  // ...
};
int S::i = 0;
  
extern void store_in_some_buffer(void *buffer, size_t offset, int val);
  
void f() {
  store_in_some_buffer(&S::i, 0, 42);
}

{% endhighlight %}


# 风险评估

向offsetof()宏传入一个无效的类型或成员会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior),有可能被[利用](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-exploit)，导致数据完整性被破坏，或者在宏展开时导致不正确的值。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP59-CPP|中|不太可能|中|P4|L3|

# 参考书目

|[ISO/IEC 9899:1999](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC9899-1999)|Subclause 7.17, "Common Definitions <stddef.h>"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 9.4.2, "Static Data Members"|
||Subclause 18.2, "Types"|

# 参考

[EXP59-CPP. Use offsetof() on valid types and members][1]

[offsetof][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP59-CPP.+Use+offsetof%28%29+on+valid+types+and+members

[2]: https://en.cppreference.com/w/cpp/types/offsetof

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。
