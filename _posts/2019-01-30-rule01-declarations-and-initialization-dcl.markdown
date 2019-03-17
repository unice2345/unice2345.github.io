---
layout: post
title: "规则01-声明和初始化(DCL)"
date:   2019-01-30 07:15:54 +0800
categories: jekyll update
---

DCL50-CPP. Do not define a C-style variadic function

DCL51-CPP. Do not declare or define a reserved identifier

DCL52-CPP. Never qualify a reference type with const or volatile

DCL53-CPP. Do not write syntactically ambiguous declarations

DCL54-CPP. Overload allocation and deallocation functions as a pair in the same scope

DCL55-CPP. Avoid information leakage when passing a class object across a trust boundary

DCL56-CPP. Avoid cycles during initialization of static objects

DCL57-CPP. Do not let exceptions escape from destructors or deallocation functions

DCL58-CPP. Do not modify the standard namespaces

DCL59-CPP. Do not define an unnamed namespace in a header file

DCL60-CPP. Obey the one-definition rule

下面这些[SEI CERT C Coding Stardard](https://wiki.sei.cmu.edu/confluence/display/c/SEI+CERT+C+Coding+Standard)的规则也适用于C++代码:

[DCL30-C. Declare objects with appropriate storage durations](https://wiki.sei.cmu.edu/confluence/display/c/DCL30-C.+Declare+objects+with+appropriate+storage+durations)

[DCL39-C. Avoid information leakage when passing a structure across a trust boundary](https://wiki.sei.cmu.edu/confluence/display/c/DCL39-C.+Avoid+information+leakage+when+passing+a+structure+across+a+trust+boundary)

[DCL40-C. Do not create incompatible declarations of the same function or object](https://wiki.sei.cmu.edu/confluence/display/c/DCL40-C.+Do+not+create+incompatible+declarations+of+the+same+function+or+object)

# 风险评估总结

|规则|严重性|可能性|修复成本|优先级|等级|
|----|----|----|----|----|----|
|DCL50-CPP|高|有可能|中|P12|L1|
|DCL51-CPP|低|不太可能|低|P3|L3|
|DCL52-CPP|低|不太可能|低|P3|L3|
|DCL53-CPP|低|不太可能|中|P2|L3|
|DCL54-CPP|低|有可能|低|P6|L2|
|DCL55-CPP|低|不太可能|高|P1|L3|
|DCL56-CPP|低|不太可能|中|P2|L3|
|DCL57-CPP|低|非常可能|中|P6|L2|
|DCL58-CPP|高|不太可能|中|P6|L2|
|DCL59-CPP|中|不太可能|中|P4|L3|
|DCL60-CPP|高|不太可能|高|P3|L3|


# 参考链接

[Rule 01. Declarations and Initialization (DCL)][1]

[1]: https://wiki.sei.cmu.edu/confluence/pages/viewpage.action?pageId=88046322
