---
layout: post
title: "Markdown学习"
date:   2019-01-01 08:26:54 +0800
categories: jekyll update
---

# 1.标题

## 1.2 二级标题

### 1.2.1 三级标题

采用一个或多个#来设置一级或多级标题

多级标题好像不起作用???

# 2.段落
用空行来分隔段落(效果看起来是这样的)

一个段落

另一个段落


一个稍微远一些的段落（看来一个和多个空行所起的作用是一样的）

行末两个空格  产生换行  

# 3.区块引用
使用>生成区块引用。多个>可以生成嵌套的区块引用。

```
>区块1
>>区块2
>>>区块3
```

>区块1
>>区块2
>>>区块3

我还感受不到这个功能有什么作用。

# 4.代码块
每行的行首加4个空格或制表符，生成代码块

    #include <iostream>
    int main() {
        std::cout << "hello,world" << std::endl;
    }

感觉书写起来不是太方便 

也可以用\`\`\`来标记代码段：

```
#include <iostream>
int main() {
    std::cout << "hello,world" << std::endl;
}
```

jekyll内部支持的代码块格式书写起来比较方便:

{% highlight c++ %}
#include <iostream>:
int main() {
    std::cout << "hello,world" << std::endl;
}
{% endhighlight %}

# 5.强调
采用\*或\_实现对文字显示成斜体

采用\*\*或\_\_实现对文字显示成粗体

例如: *强调1* _强调2_ **强调3**  __强调4__

# 6.列表

## 6.1 无序列表

采用\.或\+或-符号构建无序列表

列表内容要和无序列表符号之前有空格或制表符

+ 第一项
+ 第二项
+ 第三项

## 6.2 有序列表

采用数字+\.生成有序列表

1. 第一项
2. 第二项
3. 第三项

# 7. 分割线

采用三个\*或\-或\_

***

# 8. 反斜杠\\

使符号变成普通文本来显示

# 9. 符号\`

用来做标记

`ctrl + a`


# 10. 链接

行内式
```
[Markdown基本语法](https://github.com/younghz/Markdown)
```
[Markdown基本语法](https://github.com/younghz/Markdown)

参考式
```
[Markdown基本语法][1]
[1]: https://github.com/younghz/Markdown
```
[Markdown基本语法][1]

[1]: https://github.com/younghz/Markdown

# 11. 脚注
```
这是一个有脚注的句子[^1]
[^1]: 这是一个脚注
```

这是一个有脚注的句子[^1]

[^1]: 这是一个脚注

# 12. 穿过文字的线

```
~~The world is flat.~~
```

~~The world is flat.~~

# 13. 图片

与链接类似，只不过需要在最前面添加\!

```
这是一个图片![jekyll图标](/assets/logo-2x.png)
```

这是一个图片![jekyll图标](/assets/logo-2x.png)


# 14. 文件

与链接的格式一致

```
这是一个文件[sei-cert-c++.pdf](/assets/sei-cert-cpp-coding-standard-2016-v01.pdf)
```

这是一个文件[sei-cert-c++.pdf](/assets/sei-cert-cpp-coding-standard-2016-v01.pdf)

# 15. 表格

```
|列1| 列2 |
|----|----|
| 元素 | 元素 |
| 元素 | 元素 |
```

|列1| 列2 |
|----|----|
| 元素 | 元素 |
| 元素 | 元素 |

# 参考链接:

[Markdown基本语法][1]

[维基百科上的Markdown条目][2]

[Markdown指导][3]

[John Gruber撰写的Markdown说明][4]

[1]: https://github.com/younghz/Markdown
[2]: https://zh.wikipedia.org/wiki/Markdown
[3]: https://www.markdownguide.org/
[4]: https://daringfireball.net/projects/markdown/