---
layout: post
title:  "DCL55-CPP 当跨越信任边界传递类对象时避免信息泄露"
date:   2019-03-05 06:26:54 +0800
categories: jekyll update
---

C++标准，[class.mem]，第13段 \[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)\]描述了一个非联合类的非静态成员数据的内存布局，说明如下：

> 一个类(非联合)的具有同样访问控制权限的非静态数据成员，在类对象内部，后分配的成员具有高地址。
> 不同访问控制权限的非静态数据成员的内存分配顺序是未指定的。
> 实现上的内存对齐要求可能会导致两个相邻的成员在内存上并不是相邻的，管理虚函数和虚基类也可能需要一些空间。

进一步，[class.bit]，第1段，部分阐述如下：

> 类对象内的位字段(bit-fields)的内存分配取决于实现。位字段的内存对齐取决于实现。
> 位字段被打包填充到某个可访问的内存分配单元。

所以，填充位(padding bits)可能位于类对象示例中的任意位置（包括位于对象的起始处，比如类中声明的第一个成员是一个未命名的位字段）。
除非通过置零初始化，填充位可能包含[不确定的值](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-indeterminatevalue)，其中包含着敏感信息。


# 不遵从规范的示例代码

这个不遵从规范的示例代码运行在内核空间并将数据从arg拷贝到用户空间。然后，对象中可能使用了填充位，比如，为了实现类的数据成员的对齐。这些填充位可能包含敏感信息，当数据拷贝到用户空间时会泄露出去，不管数据是如何拷贝的。

{% highlight c++ %}
#include <cstddef>
  
struct test {
  int a;
  char b;
  int c;
};
  
// Safely copy bytes to user space
extern int copy_to_user(void *dest, void *src, std::size_t size);
  
void do_stuff(void *usr_buf) {
  test arg{1, 2, 3};
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
{% endhighlight %}


# 不遵从规范的示例代码

在这个不遵从规范的示例代码中，arg通过直接初始化方式进行了初始化。因为test没有用户提供的默认构造函数，在值初始化之前先调用了零初始化([2],[3])，保证了进一步初始化之前所有的填充位都被初始化成了0值。这相当于使用std::memset()把对象中所有的位都设置为0值。

{% highlight c++ %}
#include <cstddef>
  
struct test {
  int a;
  char b;
  int c;
};
  
// Safely copy bytes to user space
extern int copy_to_user(void *dest, void *src, std::size_t size);
  
void do_stuff(void *usr_buf) {
  test arg{};
  
  arg.a = 1;
  arg.b = 2;
  arg.c = 3;
  
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
{% endhighlight %}

但是，编译器可以任意实现arg.b = 2，可以把32位寄存器的低bytes设置成2，高bytes不修改，
然后把寄存器的全部32位值拷贝到内存。这样会把寄存器中高bytes的信息泄露给用户。


# 遵从规范的示例代码

这个遵从规范的解决方案把结构体中的数据先进行了序列化，然后再拷贝到非信任的上下文中。

{% highlight c++ %}
#include <cstddef>
#include <cstring>
  
struct test {
  int a;
  char b;
  int c;
};
  
// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, std::size_t size);
  
void do_stuff(void *usr_buf) {
  test arg{1, 2, 3};
  // May be larger than strictly needed.
  unsigned char buf[sizeof(arg)];
  std::size_t offset = 0;
  
  std::memcpy(buf + offset, &arg.a, sizeof(arg.a));
  offset += sizeof(arg.a);
  std::memcpy(buf + offset, &arg.b, sizeof(arg.b));
  offset += sizeof(arg.b);
  std::memcpy(buf + offset, &arg.c, sizeof(arg.c));
  offset += sizeof(arg.c);
  
  copy_to_user(usr_buf, buf, offset /* size of info copied */);
}
{% endhighlight %}

这段代码保证了没有未初始化的填充位被拷贝给无权限的用户。
拷贝到用户空间的结构体现在是一个打包的结构体，copy_to_user()函数之后需要将结构体解包，
重新恢复成原来的结构体。


# 遵从规范的示例代码（填充字节）

填充位可以显示声明为结构体中的字段。这个解决方案是不可移植的，因为它取决于实现和目标内存架构。
下面的这个解决方案只适用于x86-32架构。

{% highlight c++ %}
#include <cstddef>
 
struct test {
  int a;
  char b;
  char padding_1, padding_2, padding_3;
  int c;
  
  test(int a, char b, int c) : a(a), b(b),
    padding_1(0), padding_2(0), padding_3(0),
    c(c) {}
};
// Ensure c is the next byte after the last padding byte.
static_assert(offsetof(test, c) == offsetof(test, padding_3) + 1,
              "Object contains intermediate padding");
// Ensure there is no trailing padding.
static_assert(sizeof(test) == offsetof(test, c) + sizeof(int),
              "Object contains trailing padding");
 
 
 
// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, std::size_t size);
 
void do_stuff(void *usr_buf) {
  test arg{1, 2, 3};
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
{% endhighlight %}


static_assert()声明接受一个常量表达式和一个[出错信息](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-error).
表达式在编译期间被评估，如果是false，那么编译会终止并将错误信息输出为诊断信息。
在结构体中显示地插入填充字节应该保证编译器不会填充更多的字节，
所以两个static assertions应该都是true.
需要验证这些假设来保证这个方案对某种特别的实现也是正确的。


# 不遵从规范的示例代码

在这个不遵从规范的示例代码中，填充位可能比较多，包括:

+ 虚函数表或虚基类数据后的对齐填充位，用来对齐随后的数据成员
+ 用来将随后的数据成员放置到合适的对齐边界的对齐填充位
+ 用来放置不同访问控制级别的数据成员的对齐填充位
+ 当连续的位字段没有填满一整个内存分配单元时，用来补齐的位字段填充位
+ 当两个相邻的位字段声明为不同的类型时，用来补齐的位字段填充位
+ 当声明的位字段的长度比相关的内存单元的位长度还要长时，用来补齐的填充位
+ 在一个列表中保证类实例被合适的对齐的填充位

这段代码示例运行在内核空间，并将数据从arg拷贝到用户空间。然而，对象实例中的填充位可能包含敏感信息，当数据拷贝到用户空间时会被泄露。

{% highlight c++ %}
#include <cstddef>
 
class base {
public:
  virtual ~base() = default;
};
 
class test : public virtual base {
  alignas(32) double h;
  char i;
  unsigned j : 80;
protected:
  unsigned k;
  unsigned l : 4;
  unsigned short m : 3;
public:
  char n;
  double o;
   
  test(double h, char i, unsigned j, unsigned k, unsigned l, unsigned short m,
       char n, double o) :
    h(h), i(i), j(j), k(k), l(l), m(m), n(n), o(o) {}
   
  virtual void foo();
};
 
// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, std::size_t size);
 
void do_stuff(void *usr_buf) {
  test arg{0.0, 1, 2, 3, 4, 5, 6, 7.0};
  copy_to_user(usr_buf, &arg, sizeof(arg));
}
{% endhighlight %}


填充位是取决于实现的，所以类对象的内存布局在不同的编译器和架构上可能不同。
当在x86-32架构上使用[GCC](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-gcc) 5.3.0编译时，test对象需要96字节的空间，
其中29字节是数据，33字节被包含在vtable中，它的内存布局如下：

|偏移量(字节(位数))|占用内存大小(字节(位数))|原因||偏移量|占用内存大小|原因|
|----|----|----|----|----|----|----|
|41 (328)|3(24)|数据成员对齐填充||61(488)|1(8)|char n|
|54 (432)|2(16)|对齐填充||72(576)|24(192)|类对齐填充|
|44 (352)|4(32)|unsigned j: 80||62(496)|2(16)|数据成员填充|
|48 (384)|6(48)|扩展的位字段大小填充||64(512)|8(64)|double o|
|0|1(32)|vtable指针||56(448)|4(32)|unsigned k|
|4 (32)|28(224)|数据成员对齐填充||60(480)|0(4)|unsigned l:4|
|32 (256)|8(64)|double h||60(484)|0(3)|unsigned short m: 3|
|40 (320)|1(8)|char i||60(487)|0(1)|未使用的位字段中的位|


# 遵从规范的解决方案

由于数据结构比较复杂，所以这个遵从规范的解决方案将对象数据进行了序列化，然后再拷贝到不信任的上下文环境中。没有采用手动填充字节的方法。

{% highlight c++ %}
#include <cstddef>
#include <cstring>
  
class base {
public:
  virtual ~base() = default;
};
class test : public virtual base {
  alignas(32) double h;
  char i;
  unsigned j : 80;
protected:
  unsigned k;
  unsigned l : 4;
  unsigned short m : 3;
public:
  char n;
  double o;
   
  test(double h, char i, unsigned j, unsigned k, unsigned l, unsigned short m,
       char n, double o) :
    h(h), i(i), j(j), k(k), l(l), m(m), n(n), o(o) {}
   
  virtual void foo();
  bool serialize(unsigned char *buffer, std::size_t &size) {
    if (size < sizeof(test)) {
      return false;
    }
     
    std::size_t offset = 0;
    std::memcpy(buffer + offset, &h, sizeof(h));
    offset += sizeof(h);
    std::memcpy(buffer + offset, &i, sizeof(i));
    offset += sizeof(i);
    unsigned loc_j = j; // Only sizeof(unsigned) bits are valid, so this is not narrowing.
    std::memcpy(buffer + offset, &loc_j, sizeof(loc_j));
    offset += sizeof(loc_j);
    std::memcpy(buffer + offset, &k, sizeof(k));
    offset += sizeof(k);
    unsigned char loc_l = l & 0b1111;
    std::memcpy(buffer + offset, &loc_l, sizeof(loc_l));
    offset += sizeof(loc_l);
    unsigned short loc_m = m & 0b111;
    std::memcpy(buffer + offset, &loc_m, sizeof(loc_m));
    offset += sizeof(loc_m);
    std::memcpy(buffer + offset, &n, sizeof(n));
    offset += sizeof(n);
    std::memcpy(buffer + offset, &o, sizeof(o));
    offset += sizeof(o);
     
    size -= offset;
    return true;
  }
};
  
// Safely copy bytes to user space.
extern int copy_to_user(void *dest, void *src, size_t size);
  
void do_stuff(void *usr_buf) {
  test arg{0.0, 1, 2, 3, 4, 5, 6, 7.0};
   
  // May be larger than strictly needed, will be updated by
  // calling serialize() to the size of the buffer remaining.
  std::size_t size = sizeof(arg);
  unsigned char buf[sizeof(arg)];
  if (arg.serialize(buf, size)) {
    copy_to_user(usr_buf, buf, sizeof(test) - size);
  } else {
    // Handle error
  }
}
{% endhighlight %}

这段代码保证了没有未初始化的填充位被拷贝到非授权的用户。被拷贝到用户空间的数据结构是一个打包的数据结构，copy_to_user()函数需要进行解包来创建原始的填充的数据结构。

# 风险评估

填充位可能会无意地包含敏感数据，比如指向内核数据结构的指针或者密码。一个指向这样一个数据结构的指针有可能被传递给其他函数，造成信息泄露。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL55-CPP|低|不太可能|高|P1|L3|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard) |[DCL39-C. Avoid information leakage when passing a structure across a trust boundary ](https://wiki.sei.cmu.edu/confluence/display/c/DCL39-C.+Avoid+information+leakage+when+passing+a+structure+across+a+trust+boundary)|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 8.5, "Initializers"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 9.2, "Class Members"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 9.6, "Bit-fields" |


# 参考链接

[DCL55-CPP. Avoid information leakage when passing a class object across a trust boundary][1]

[zero initialization][2]

[value initialization][3]

[alignas specifier][4]

[Bit field][5]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL55-CPP.+Avoid+information+leakage+when+passing+a+class+object+across+a+trust+boundary

[2]: https://en.cppreference.com/w/cpp/language/zero_initialization

[3]: https://en.cppreference.com/w/cpp/language/value_initialization

[4]: https://en.cppreference.com/w/cpp/language/alignas

[5]: https://en.cppreference.com/w/cpp/language/bit_field

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。