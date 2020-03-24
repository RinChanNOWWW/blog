---
title: 是否是数字
date: 2020-03-24 20:31:38
tags:
	- 算法题
	- 状态机
	- Python
categories:
	- 算法题
---
# 是否是数字

一道很恶心的题。如果真的是笔试时候不知道自己哪种例子没过就凉凉。

> 请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串"+100","5e2","-123","3.1416"和"-1E-16"都表示数值。 但是"12e","1a3.14","1.2.3","+-5"和"12e+4.3"都不是。

遇到的坑：

-  **小数点前可以什么都没有**。+.123 这种也算做数字，也可以添加正负号。
-  **e/E 后面必须有数字**

Python 没有 `switch` 那就暴力 `if-else` 了。（其实我想用 C++ 的，但用都用 Python 了就继续吧。

```python
def isNumeric(s):
    # write code here
    if len(s) == 0 or s == '+' or s == '-' or s == 'e' or s == 'E' or s == '.':
        return False
    numbers = ['0','1','2','3','4','5','6','7','8','9']
    signs = ['+', '-']
    e = ['e', 'E']
    
    if s[0] not in numbers + signs:
        return False
    if s[0] in signs:
        state = 0
    else:
        state = 1
    
    for i in range(1, len(s)):
        
        if state == 0:
            if s[i] in numbers + ['.']:
                if s[i] in numbers:
                    state = 1
                else:
                    state = 2
            else:
                return False
        
        elif state == 1:
            if s[i] in numbers + e + ['.']:
                if s[i] == '.':
                    state = 2
                elif s[i] in e:
                    if i == len(s) - 1:
                        return False
                    state = 3
            else:
                return False
            
        elif state == 2:
            if s[i] in numbers:
                state = 5
            else:
                return False
            
        elif state == 3:
            if s[i] in signs + numbers:
                state = 4
            else:
                return False
            
        elif state == 4:
            if s[i] not in numbers:
                return False

        elif state == 5:
            if s[i] in numbers + e:
                if s[i] in e:
                    state = 3
            else:
                return False

    return True
```
