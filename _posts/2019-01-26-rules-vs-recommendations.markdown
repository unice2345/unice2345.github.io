---
layout: post
title:  "规则 vs 建议"
date:   2019-1-16 08:26:54 +0800
categories: jekyll update
---

CERT编码规范由*规则*和*建议*组成。
规则指的是对代码的规范性要求。
建议指的是如果遵守了这样的指导，能够改善软件系统的安全性、[可靠性][reliability]和可防御性。
但是，如果不遵守建议，也并不意味着代码中一定会引入缺陷。
规则和建议统称为*指南*。

# 规则
规则必须满足以下标准:
1. 违反这些指南易导致代码缺陷，对软件系统的安全性、可靠性和可防御性造成不利影响。
比如，引入了一个[安全缺陷][security flaw]，导致可利用的[漏洞][vulnerability]。
2. 这些指南并不依赖代码注释和假设。
3. 可以通过自动化分析(静态或动态的)、形式化方法或手动检查等手段检测是否遵守了这些指南。

# 建议
建议指的是能提升代码质量的参考性意见。满足以下条件的指南称为建议：
1. 采用这些指南能够提升软件系统的安全性、可靠性和可防御性。
2. 有些指南可能不满足规则的条件，被归为建议。

开发过程中要采用哪些建议取决于最终的软件产品。一些严格要求的项目可能决定投入更多的资源去提升系统的安全性、
可靠性和可防御性，那么就会采用更广范围的建议。

目前CERT C++编程规范不包括任何建议；所有的建议都被移除了，等待未来进行评审和开发。

# 参考链接

[Rules Versus Recommendations][1]

[1]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/Rules+Versus+Recommendations
[reliability]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-reliability
[security flaw]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-securityflaw
[vulnerability]: https://wiki.sei.cmu.edu/confluence/display/cplusplus/BB.+Definitions#BB.Definitions-vulnerability
