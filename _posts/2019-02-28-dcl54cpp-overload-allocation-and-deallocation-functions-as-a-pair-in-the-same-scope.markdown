---
layout: post
title:  "DCL54-CPP 在同一个作用域内要成对地重载内存分配函数和解分配函数"
date:   2019-02-28 06:26:54 +0800
categories: jekyll update
---

内存分配函数和解分配函数可以在全局作用域和类作用域进行重载。

在一个特定的作用域内，如果一个内存分配函数被重载，那么对应的内存解分配函数必须在相同的作用域内被重载。（反过来也一样）

没有重载相应的动态内存函数易导致违反类似[MEM51-CPP. Properly deallocate dynamically allocated resources](https://wiki.sei.cmu.edu/confluence/display/cplusplus/MEM51-CPP.+Properly+deallocate+dynamically+allocated+resources)的规则。
比如，如果一个重载的内存分配函数使用私有的堆空间实施了一次内存分配，并把返回的指针传给了默认的内存解分配函数，容易导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior)。
即使重载的内存分配函数最终使用的是默认的内存分配器获取的内存，没有重载相应的内存解分配函数时，还是有可能由于没有更新自定义的分配器的内部数据，使程序处于非预期的状态。


可以将内存分配和解分配函数定义为[被删除的函数][4], 不使用对应的[自由存储](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-freestore)函数来实现。
比如，当类中已经定义了一个占位形式的new函数时(placement new function)时，一种常见的惯例是同时定义删除的非占位形式的内存分配和解分配成员函数。
这样可以阻止不小心通过new调用该类的内存分配函数，和通过delete调用该类的内存解分配函数。
基于同样的原因，可以声明但不定义私有的内存分配或解分配函数。
但是，不能提供定义，因为那样仍然会在成员函数内部允许访问到自由存储函数(???)。


# 不遵从规范的示例代码

在这个不遵从规范的示例代码中，一个全局作用域的内存分配函数被重载了。
但是，对应的内存解分配函数却没有被声明。
对一个通过重载的内存分配函数创建的对象，任何对该对象进行delete都会导致[未定义的行为](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-undefinedbehavior), 因为违反了规则[MEM51-CPP. Properly deallocate dynamically allocated resources](https://wiki.sei.cmu.edu/confluence/display/cplusplus/MEM51-CPP.+Properly+deallocate+dynamically+allocated+resources)。

{% highlight c++ %}
#include <Windows.h>
#include <new>
  
void *operator new(std::size_t size) noexcept(false) {
  static HANDLE h = ::HeapCreate(0, 0, 0); // Private, expandable heap.
  if (h) {
    return ::HeapAlloc(h, 0, size);
  }
  throw std::bad_alloc();
}
  
// No corresponding global delete operator defined.
{% endhighlight %}


# 遵从规范的示例代码

在这个遵从规范的解决方案中，对应的内存解分配函数也被定义在全局作用域。

{% highlight c++ %}
#include <Windows.h>
#include <new>
 
class HeapAllocator {
  static HANDLE h;
  static bool init;
  
public:
  static void *alloc(std::size_t size) noexcept(false) {
    if (!init) {
      h = ::HeapCreate(0, 0, 0); // Private, expandable heap.
      init = true;
    }
  
    if (h) {
      return ::HeapAlloc(h, 0, size);
    }
    throw std::bad_alloc();
  }
  
  static void dealloc(void *ptr) noexcept {
    if (h) {
      (void)::HeapFree(h, 0, ptr);
    }
  }
};
  
HANDLE HeapAllocator::h = nullptr;
bool HeapAllocator::init = false;
 
void *operator new(std::size_t size) noexcept(false) {
  return HeapAllocator::alloc(size);
}
  
void operator delete(void *ptr) noexcept {
  return HeapAllocator::dealloc(ptr);
}
{% endhighlight %}


# 不遵从规范的示例代码

在这个不遵从规范的示例代码中，operator new()在类作用域内被重载了，但是operator delete()却没有在类作用域内相应地重载。
尽管重载的内存分配函数也是调用默认的全局内存分配函数，但是当一个类型S的对象被分配后，
任何对该对象的delete都会导致程序处于不确定的状态，因为没有对应地去更新分配的bookkeeping。

{% highlight c++ %}
#include <new>
  
extern "C++" void update_bookkeeping(void *allocated_ptr, std::size_t size, bool alloc);
  
struct S {
  void *operator new(std::size_t size) noexcept(false) {
    void *ptr = ::operator new(size);
    update_bookkeeping(ptr, size, true);
    return ptr;
  }
};
{% endhighlight %}


# 遵从规范的示例代码

在这个遵从规范的解决方案中，对应的operator delete()在相同的类作用域内被重载了。

{% highlight c++ %}
#include <new>
 
extern "C++" void update_bookkeeping(void *allocated_ptr, std::size_t size, bool alloc);

struct S {
  void *operator new(std::size_t size) noexcept(false) {
    void *ptr = ::operator new(size);
    update_bookkeeping(ptr, size, true);
    return ptr;
  }
  
  void operator delete(void *ptr, std::size_t size) noexcept {
    ::operator delete(ptr);
    update_bookkeeping(ptr, size, false);
  }
};
{% endhighlight %}


# 例外

**DCL54-CPP-EX1:** 一个占位形式的内存解分配函数可以被省略，但是只有当对应的对象占位内存分配函数和对象构造器保证是noexcept(true)时才行。
因为当对象初始化由于抛出一个异常而中止的时候，占位形式的内存解分配函数会被自动调用，所以当异常不能被抛出时，省略占位形式的内存解分配函数是安全的。
比如，一些厂商实现的编译器标志禁止异常支持（比如像[Clang](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-clang)中的-fno-cxx-exceptions和[Microsoft Visual Studio](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-msvc)中的/EHs-c-），当抛出异常时结果是[行为取决于实现](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-implementation-definedbehavior), 一般的结果是程序中止，类似调用abort()。

# 风险评估

new和delete不匹配可能导致[拒绝服务攻击](https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-denial-of-service)。

|规则|严重性|可能性|修复代价|优先级|等级|
|--|--|--|--|--|--|
|DCL54-CPP|低|一般可能|低|P6|L2|

**自动检测**

略

**相关漏洞**

略

# 相关指南

|[SEI CERT C++ Coding Standard](https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046682)|[MEM51-CPP. Properly deallocate dynamically allocated resources](https://wiki.sei.cmu.edu/confluence/display/cplusplus/MEM51-CPP.+Properly+deallocate+dynamically+allocated+resources)|

# 参考书目

|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 3.7.4, "Dynamic Storage Duration"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 5.3.4, "New"|
|[ISO/IEC 14882-2014](https://wiki.sei.cmu.edu/confluence/display/cplusplus/AA.+Bibliography#AA.Bibliography-ISO/IEC14882-2014)|Subclause 5.3.5, "Delete" |



# 参考链接

[DCL54-CPP. Overload allocation and deallocation functions as a pair in the same scope][1]

[operator new][2]

[operator delete][3]

[Deleted functions][4]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/DCL54-CPP.+Overload+allocation+and+deallocation+functions+as+a+pair+in+the+same+scope

[2]: https://en.cppreference.com/w/cpp/memory/new/operator_new

[3]: https://en.cppreference.com/w/cpp/memory/new/operator_delete

[4]: https://en.cppreference.com/w/cpp/language/function#Deleted_functions

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。