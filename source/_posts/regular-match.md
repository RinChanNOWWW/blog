---
title: 正则匹配
date: 2020-03-24 16:44:19
tags: 算法题
categories: 
	- 算法题
---
# 正则匹配

这题每次做的时候都想半天，记录一下。

可以使用递归或者动规的方法。

> 请实现一个函数用来匹配包括'.'和'\*'的正则表达式。模式中的字符'.'表示任意一个字符，而'\*'表示它前面的字符可以出现任意次（包含0次）。 在本题中，匹配是指字符串的所有字符匹配整个模式。例如，字符串"aaa"与模式"a.a"和"ab\*ac\*a"匹配，但是与"aa.a"和"ab\*a"均不匹配。

**本算法的正则匹配是贪婪匹配**

```cpp
bool match(char* str, char* pattern)
{
    if (*str == '\0' && *pattern == '\0')
        return true;
    if (*pattern == '\0')
        return false;
    
    if (*(pattern + 1) != '*') {
        if (*str == *pattern || (*str != '\0' && *pattern == '.'))
            return match(str + 1, pattern + 1);
        else
            return false;
    }
    else {
        // * 匹配 0 个或多个
        if (*str == *pattern || (*str != '\0' && *pattern == '.'))
            return match(str + 1, pattern) || match(str, pattern + 2);
        else
            return match(str, pattern + 2);
   }
}

```
