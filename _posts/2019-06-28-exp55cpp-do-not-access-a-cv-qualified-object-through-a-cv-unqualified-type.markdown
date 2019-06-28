---
layout: post
title:  "EXP55-CPP 不要通过非cv修饰的类型访问cv修饰的对象"
date:   2019-06-28 06:26:54 +0800
categories: jekyll update
---

C++标准， [dcl.type.cv]，第4段  [[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)]，描述如下：

> Except that any class member declared mutable can be modified, any attempt to modify a const object during its lifetime results in undefined behavior.

类似的，第6段描述如下：

> What constitutes an access to an object that has volatile-qualified type is implementation-defined. If an attempt is made to refer to an object defined with a volatile-qualified type through the use of a glvalue with a non-volatile-qualified type, the program behavior is undefined.

不要转换掉const修饰符去试图修改对象。
const修饰符意味着API的设计者不想对象被修改，即使存在对象可以被修改的可能性。
不要转换掉volatile修饰符；
volatile修饰符意味着API设计者会以编译器不知道的方式去访问对象，
（转换掉volatile修饰符后）对volatile对象的任何访问都会导致未定义的行为。

# 不符合规范的代码示例 

在这个不符合规范的代码示例中，函数g()被传入一个const int&，然后被转换成int&并且被修改了。
因为引用值之前被声明为const， 赋值操作会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
void g(const int &ci) {
  int &ir = const_cast<int &>(ci);
  ir = 42;
}
 
void f() {
  const int i = 4;
  g(i);
}
{% endhighlight %}


# 符合规范的解决方案 

在这个符合规范的解决方案中，函数g()被传入一个int &, 调用者被要求传入一个int并且可以被修改。

{% highlight c++ %}
void g(int &i) {
  i = 42;
}
 
void f() {
  int i = 4;
  g(i);
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，一个const修饰的方法被调用，该方法试图通过转换掉this的const修饰符来修改缓存的结果。因为s已经被声明为const,修改cachedValue会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
#include <iostream>
  
class S {
  int cachedValue;
   
  int compute_value() const;  // expensive
public:
  S() : cachedValue(0) {}
   
  // ... 
  int get_value() const {
    if (!cachedValue) {
      const_cast<S *>(this)->cachedValue = compute_value(); 
    }       
    return cachedValue;
  }
};
 
void f() {
  const S s;
  std::cout << s.get_value() << std::endl;
}
{% endhighlight %}


# 符合规范的解决方案 

这个符合规范的解决方案在声明cachedValue使用了mutable关键字，允许在const上下文环境下修改cachedValue，而不会触发[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
#include <iostream>
  
class S {
  mutable int cachedValue;
   
  int compute_value() const;  // expensive
public:
  S() : cachedValue(0) {}
   
  // ... 
  int get_value() const {
    if (!cachedValue) {
      cachedValue = compute_value(); 
    }       
    return cachedValue;
  }
};
 
void f() {
  const S s;
  std::cout << s.get_value() << std::endl;
}
{% endhighlight %}


# 不符合规范的代码示例 

在这个不符合规范的代码示例中，易变值s的volatile修饰符被转换掉了，
在g()中试图读取该值会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。

{% highlight c++ %}
#include <iostream>
 
struct S {
  int i;
   
  S(int i) : i(i) {}
};
 
void g(S &s) {
  std::cout << s.i << std::endl;
}
 
void f() {
  volatile S s(12);
  g(const_cast<S &>(s));
}
{% endhighlight %}


# 符合规范的解决方案 

这个符合规范的解决方案假定s要求易变性，所以g()被修改成接受一个volatile S&类型的参数。

{% highlight c++ %}

#include <iostream>
 
struct S {
  int i;
   
  S(int i) : i(i) {}
};
 
void g(volatile S &s) {
  std::cout << s.i << std::endl;
}
 
void f() {
  volatile S s(12);
  g(s);
}
{% endhighlight %}


# 例外

**EXP55-CPP-EX1:** 这条规则的一个例外情况是，当调用一个不接受const参数的遗留的API时允许转换掉const，
前提是函数不会修改引用的变量值。
然而，在可能的情况下，还是应该将API修改成const正确的形式。
例如，下面的代码在调用audit_log()函数时转换掉了INVFNAME的const修饰。

{% highlight c++ %}
// Legacy function defined elsewhere - cannot be modified; does not attempt to
// modify the contents of the passed parameter.
void audit_log(char *errstr);
 
void f() {
  const char INVFNAME[]  = "Invalid file name.";
  audit_log(const_cast<char *>(INVFNAME));
}
{% endhighlight %}

# 风险评估

如果一个对象被声明为不可变的，它在运行时有可能位于写保护的内存空间。
试图修改这样一个对象可能会导致[程序异常终止](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-abnormaltermination)或[拒绝服务攻击](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-denial-of-service)。
如果一个对象被声明为可变的，那么编译器不会对该对象的访问做出假定。
转换掉一个对象的可变性会导致对该对象的重写被重排顺序或删除掉，导致程序执行异常。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP55-CPP|中|有可能|中|P8|L2|


# 相关规则

|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[EXP32-C. Do not access a volatile object through a nonvolatile reference](https://wiki.sei.cmu.edu/confluence/display/c/EXP32-C.+Do+not+access+a+volatile+object+through+a+nonvolatile+reference)|
||[EXP40-C. Do not modify constant objects ](https://wiki.sei.cmu.edu/confluence/display/c/EXP40-C.+Do+not+modify+constant+objects)|


# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 7.1.6.1, "The cv-qualifiers"|
|[Sutter 2004](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-Sutter04)| Item 94, "Avoid Casting Away const" |


# 参考

[EXP55-CPP. Do not access a cv-qualified object through a cv-unqualified type][1]

[cv (const and volatile) type qualifiers][2]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP55-CPP.+Do+not+access+a+cv-qualified+object+through+a+cv-unqualified+type

[2]: https://en.cppreference.com/w/cpp/language/cv


<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。