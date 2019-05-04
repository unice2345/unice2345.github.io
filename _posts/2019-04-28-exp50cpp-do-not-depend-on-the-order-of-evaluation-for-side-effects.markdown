---
layout: post
title:  "EXP50-CPP 不要依赖求值顺序来处理副作用"
date:   2019-04-28 07:10:54 +0800
categories: jekyll update
---

在C++中，修改一个对象，调用一个库I/O函数，访问一个volatile修饰的值，或者调用一个函数执行以上动作都是修改执行环境状态的方法。
这些动作被称为*副作用*。
所有的值计算和副作用之间的关系可以用它们的求值顺序来描述。
C++标准，\[intro.execution\]，第13段，\[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)\]建立了三种求值顺序：

> *按顺序早于*规则是同一线程中的求值之间的一种非对称的，传递的，对偶的关系，
它包含了这些求值的局部顺序。
假设有两个求值A和B，如果A的顺序早于B，那么对A的求值应该在对B的求值之前执行。
如果A的顺序不早于B，并且B的顺序也不早于A，那么A和B是*无顺序(unsequenced)*的。
[注意，对无顺序的求值，执行过程可以交叠(指令交错)]。
如果A的顺序可以早于B，B的顺序也可以早于A，但是没有指定谁早于谁，那么A和B是*顺序不确定的(indetermiantely sequenced)*。
[注意，对顺序不确定的求值，执行过程是不能交叠的，但是任意一个都可以先执行]

第15段进一步阐述如下(为了保持简洁，移除了非规范文本):

> 除非特别阐明，对单个操作符的操作数和对单个表达式的子表达式的求值是无顺序的(unsequenced)...
对一个操作符的操作数的值计算按顺序早于对该操作符的结果的值计算。
如果有一个标量对象的一项副作用相对于同一个标量对象上的另外一个副作用是无顺序的，
或者相对于使用同一个标量对象的值进行的值计算是无顺序的，而且它们不是潜在并发的，
那么行为是未定义的...当调用一个函数时(无论该函数是否是内联)，与任何实参表达式或指代被调用函数的相关的值计算和副作用，都按顺序早于被调用函数体内部的表达式和语句的执行...
在调用函数(包括其他函数调用)中的每个求值如果没有明确按顺序早于或晚于被调用函数体的执行，
那么他们和被调用函数的执行是顺序不确定的(indeterminatedly sequenced)。
C++中的几种场景下会导致函数调用的求值，即使在翻译单元中没有相应的函数调用语法出现...
在被调用函数的执行上的顺序限制(如上所述)是函数调用被计算时的特性，无论调用函数的表达式语法是什么样子。

不要让同一个标量对象出现在无顺序的或顺序不确定的操作两侧的副作用或值计算中。

下面这些表达式具有顺序限制, 这些顺序限制是从通常无顺序情况下偏离产生的[[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)]：

+ 在后缀++和--表达式中，值计算按顺序早于对操作数的修改。([expr.post.incr],第1段)
+ 在逻辑&&表达式中，如果第二个表达式被求值，所有第一个表达式相关的值计算和副作用按顺序早于所有第二个表达式的值计算和副作用。([expr.log.and],第2段)
+ 在逻辑\|\|表达式中，如果第二个表达式被求值，所有第一个表达式相关的值计算和副作用按顺序早于所有第二个表达式的值计算和副作用。([expr.log.and],第2段)
+ 在条件?:表达式中，所有第一个表达式相关的值计算和副作用按顺序早于所有第二个或第三个表达式的值计算和副作用（无论是否被求值）。([expr.cond], 第1段)
+ 在赋值表达式中(包括复合赋值)，赋值操作按顺序晚于左右操作数的值计算，并按顺序早于赋值表达式的值计算。([expre.ass],第1段)
+ 在逗号,表达式中，左侧表达式的值计算和副作用按顺序早于右侧表达式的值计算和副作用。([expr.comma], 第1段)
+ 在对初始化列表进行求值时，每个*初始化器字句(initializer-clause)*的值计算和副作用按顺序早于后续*初始化器字句*的值计算和副作用。([dcl.init.list], 第4段)
+ 当一个信号处理函数作为std::raise()的结果被执行时，信号处理函数的执行按顺序晚于std::raise()的调用，并按顺序早于它的返回。([intro.execution],第6段)
+ 在一个线程中，所有具有线程存储周期的初始化了的对象的析构按顺序早于具有静态存储周期的对象的析构。([basic.start.term],第1段)
+ 在一个*new表达式*中，分配对象的初始化按顺序早于*new表达式*的值计算。([expr.new],第18段)
+ 当一个默认构造函数被调用来初始化一个数组中的元素时，并且每个构造器至少有一个默认参数时，
由默认参数产生的临时变量的析构按顺序早于下一个元素的构造。([class.temporary],第4段)
+ 一个临时变量，如果它的生命周期没有通过绑定到一个引用上被扩展，那么它的析构按顺序早于同一个全表达式中早先创建的临时变量的析构。([class.temporary],第5段)
+ 原子内存顺序函数可以显示地指定表达式的执行顺序。([atomics.order]及[atomics.fences])

这些规则意味着下面的语句：
{% highlight c++ %}
i = i + 1;
a[i] = i;
{% endhighlight %}
具备定义的行为，如下的语句则不具有定义的行为：

{% highlight c++ %}
// 在同一个全表达式中，i被修改了2次
i = ++i + 1; 
 
// i除了被读取之外还决定哪个值被存储
a[i++] = i; 
{% endhighlight %}

在C++代码中，并不是所有的逗号实例都表示逗号运算符。
比如，一个函数调用中的多个参数中间的逗号就*不是*逗号运算符。
再比如，重载的运算符的行为就是一个函数调用，所以重载的逗号运算符的操作数就像一个函数调用的参数。

# 不遵从规范的代码示例

在这个不遵从规范的代码示例中，i以无序的方式被求值多次，所以这个表达式的行为是[未定义的](https://wiki.sei.cmu.edu/confluence/display/c/BB.+Definitions#BB.Definitions-undefinedbehavior).

{% highlight c++ %}
void f(int i, const int *b) {
  int a = i + b[++i];
  // ...
}
{% endhighlight %}

# 遵从规范的解决方案

这些例子不依赖于操作数的求值顺序，只能以一种方式被解析

{% highlight c++ %}
void f(int i, const int *b) {
  ++i;
  int a = i + b[i];
  // ...
}
{% endhighlight %}

{% highlight c++ %}
void f(int i, const int *b) {
  int a = i + b[i + 1];
  ++i;
  // ...
}
{% endhighlight %}


# 不遵从规范的代码示例

这个不遵从规范的代码示例中，调用fun()有[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/c/BB.+Definitions#BB.Definitions-undefinedbehavior),
因为参数表达式是无序的。

{% highlight c++ %}
extern void func(int i, int j);
  
void f(int i) {
  func(i++, i);
}
{% endhighlight %}

第一个(左边的)参数表达式读取i的值(为了决定被存储的值)然后修改了i。
第二个(右边的)参数表达式读取i的值，但是没有决定存储到i中的值。
这个再次尝试读取i的值会造成[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/c/BB.+Definitions#BB.Definitions-undefinedbehavior)

# 遵从规范的解决方案

当程序员的意图是使fun()的两个参数相等时，下面这个遵从规范的解决方案是合适的。

{% highlight c++ %}
extern void func(int i, int j);
  
void f(int i) {
  i++;
  func(i, i);
}
{% endhighlight %}

当程序员的意图是第二个参数比第一个参数的值大1时，下面这个遵从规范的解决方案是合适的。

{% highlight c++ %}
extern void func(int i, int j);
  
void f(int i) {
  int j = i++;
  func(j, i);
}
{% endhighlight %}

# 不遵从规范的代码示例

这个不遵从规范的代码示例和上面的例子类似。
但是，这个代码没有直接调用函数，而是调用了重载的operator<<()。
重载的操作符和函数本质上是一样的，对参数的顺序具有同样的限制规则。
这意味着操作数并不是从左到右进行求值的，而是无序的。
结果是，这个不规范的代码示例具有未定义的行为。

{% highlight c++ %}

#include <iostream>
  
void f(int i) {
  std::cout << i++ << i << std::endl;
}
{% endhighlight %}

# 遵从规范的解决方案

在这个遵从规范的解决方案中，对operator<<()进行了两次调用，保证参数按顺序打印。

{% highlight c++ %}
#include <iostream>
  
void f(int i) {
  std::cout << i++;
  std::cout << i << std::endl;
}
{% endhighlight %}

# 不遵从规范的代码示例

函数参数的求值顺序是未指定的。
这个不遵从规范的代码示例展示了[未指定的行为](https://wiki.sei.cmu.edu/confluence/display/c/BB.+Definitions#BB.Definitions-unspecifiedbehavior)
而非[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/c/BB.+Definitions#BB.Definitions-undefinedbehavior)

{% highlight c++ %}
extern void c(int i, int j);
int glob;
  
int a() {
  return glob + 10;
}
 
int b() {
  glob = 42;
  return glob;
}
  
void f() {
  c(a(), b());
}
{% endhighlight %}

a()和b()的调用顺序是未指定的；唯一肯定的是a()和b()都会在c()之前被调用。
当a()和b()计算它们的返回值时依赖共享的状态，就像例子中的那样，
那么生成的传给c()的参数可能对不同的编译器和架构来说有不同的值。

# 遵从规范的解决方案

在这个遵从规范的解决方案中，a()和b()的求值顺序被固定了，所以没有[未指定的行为](https://wiki.sei.cmu.edu/confluence/display/c/BB.+Definitions#BB.Definitions-unspecifiedbehavior)发生。

{% highlight c++ %}
extern void c(int i, int j);
int glob;
  
int a() {
  return glob + 10;
}
  
int b() {
  glob = 42;
  return glob;
}
  
void f() {
  int a_val, b_val;
  
  a_val = a();
  b_val = b();
 
  c(a_val, b_val);
}
{% endhighlight %}

# 风险评估

在一个无顺序的或顺序不确定的求值过程中，尝试修改一个对象会导致对象接受一个非预期的值，
进而导致非预期的程序行为。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|EXP50-CPP|中|有可能|中|P8|L2|

# 相关规则

|[SEI CERT C Coding Standard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)|[EXP30-C. Do not depend on the order of evaluation for side effects](https://wiki.sei.cmu.edu/confluence/display/c/EXP30-C.+Do+not+depend+on+the+order+of+evaluation+for+side+effects)|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.[INCITS 2014]Bibliography-ISO/IEC14882-2014)| Subclause 1.9, "Program Execution"|
|[MISRA 2008](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-MISRA08)|Rule 5-0-1 (Required)|


# 参考链接

[EXP50-CPP. Do not depend on the order of evaluation for side effects][1]

[Order of evaluation][2]

[求值顺序][3]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP50-CPP.+Do+not+depend+on+the+order+of+evaluation+for+side+effects

[2]: https://en.cppreference.com/w/cpp/language/eval_order

[3]: https://zh.cppreference.com/w/cpp/language/eval_order

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。