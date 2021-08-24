---
title: C++ 类型萃取
date: 2021-07-31 10:23:20
tags:
  - C++
  - 语言特性
categories:	
  - 编程学习
---

最近在网上冲浪的时候注意到了 `<cmath>` 中 `sqrt` 这个求平方根的函数。与 C 语言中的不同，C++ 将它重定义为了模板函数。在使用过程中我发现，这个函数的模板特化只能用整数类型，如果是其他类型，则会编译报错。

<!-- more -->

```cpp
#include <cmath>

int main() {
    std::sqrt<int>(1);       // ok
    std::sqrt<char>(1);      // ok
    std::sqrt<double>(1.0);  // compile error
    return 0;
}
```

g++ 编译报错：

```
$ g++ type_traits.cpp
type_traits.cpp: In function ‘int main()’:
type_traits.cpp:6:26: error: no matching function for call to ‘sqrt<double>(double)’
    6 |     std::sqrt<double>(1.0);  // compile error
      |                          ^
In file included from type_traits.cpp:1:
/usr/include/c++/9/cmath:475:5: note: candidate: ‘template<class _Tp> constexpr typename __gnu_cxx::__enable_if<std::__is_integer<_Tp>::__value, double>::__type std::sqrt(_Tp)’
  475 |     sqrt(_Tp __x)
      |     ^~~~
/usr/include/c++/9/cmath:475:5: note:   template argument deduction/substitution failed:
/usr/include/c++/9/cmath: In substitution of ‘template<class _Tp> constexpr typename __gnu_cxx::__enable_if<std::__is_integer<_Tp>::__value, double>::__type std::sqrt(_Tp) [with _Tp = double]’:
type_traits.cpp:6:26:   required from here
/usr/include/c++/9/cmath:475:5: error: no type named ‘__type’ in ‘struct __gnu_cxx::__enable_if<false, double>’

$ clang++ type_traits.cpp
type_traits.cpp:6:5: error: no matching function for call to 'sqrt'
    std::sqrt<double>(1.0);  // compile error
    ^~~~~~~~~~~~~~~~~
/usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/cmath:475:5: note: candidate template ignored: substitution failure [with _Tp = double]: no type named '__type' in '__gnu_cxx::__enable_if<false, double>'
    sqrt(_Tp __x)
    ^
1 error generated.
```

我大惊失色，C++ 也能指定模板特化类型了？遂打开源码一看：

```cpp
// cmath
template <typename _Tp>
inline _GLIBCXX_CONSTEXPR
    typename __gnu_cxx::__enable_if<__is_integer<_Tp>::__value, double>::__type
    sqrt(_Tp __x) {
    return __builtin_sqrt(__x);
}
```

我超，感觉有点牛逼，上网学习了一波，原来这种方法叫做类型萃取（type traits)，哇靠，Rust 是吧（

凭着摸鱼摸到底的精神，我通过这个源码学习了一波这个 type traits 是如何实现的，在这里借助 `sqrt` 这个函数记录一下逐步的分析过程。

## 类型萃取

注：以下内容可能由于操作系统平台不同以及编译器的不同存在些许差异，但是命名以及实现原理都大同小异。

`sqrt` 这个函数的定义乍一看很复杂，但其实十分的简单。简化之后来说就主要分为四个部分：

```cpp
// 按照 C++ 标准改写
template <typename _Tp> // 模板
typename std::enable_if<std::is_integral<_Tp>::value, double>::type // 返回值类型定义
sqrt(_Tp __x) { // 形式参数类型定义
    return __sqrt(__x); // 函数逻辑定义，调用内部 sqrt
}
```

简单看来，限定函数参数 `__x` 类型的就是这个 `_Tp` ，一定是定义返回值的那条语句的又限定了 `_Tp` 的类型取值。

```cpp
typename std::enable_if<std::is_integral<_Tp>::value, double>::type
```

按照字面上的意思来看，这个语句的意思是，返回值类型是从 `std::enable_if` 这个结构的作用域中导出，当 `std::is_integral<_Tp>` 为 true 时，导出的 type 为 double。那这个具体又是如何实现的呢？我们由外到内来分析一下这个语句。

### 1. typename

首先 `typename` 这个关键字限定了最后导出的这个 `::type` 一定是个类型，而不是一个变量。

### 2. std::enable_if

这个结构体的定义就是 type traits 的精髓了。

```cpp
// type_traits.h
template<bool B, class T = void>
struct enable_if {};
template<class T>
struct enable_if<true, T> { typedef T type; };
```

这样的定义使用了 C++ 非类型模板参数与模板偏特化的特性。`enable_if` 的第一个参数是个 `bool` 类型的非类型模板参数 B，可以被赋值。然后 `enable_if` 又被偏特化出了一个 B 为 true 的结构，此结构中定义将模板参数 T 重命名为了 type。通过这样的操作，只有 `std::enable_if<true, T>` 的时候能过够通过作用域符导出 type，并且 type 就是 T 这个类型，**如果 B == false，则不存在可以导出 type 的类型，导致类型不存在的编译错误**。

简单来说，就是 `typename std::enable_if<B, T>::type` 的作用就是当 B 为 true 时，这个语句就相当于 T，否则就是不存在的类型。

### 3. std::is_integral

了解了 `enable_if` 的作用之后就更加清晰了，`std::is_integral<_Tp>::value` 的值一定是个布尔值，那么它又是如何实现来判断 _Tp 是个整形的呢？下面列举了一个简化版的实现。

```cpp
// type_traits.h
template <typename _Tp>
struct is_integral {
    enum { value = 0 };
};
template <>
struct is_integral<int> {
    enum { value = 1 };
};
template <>
struct is_integral<bool> {
    enum { value = 1 };
};
template <>
struct is_integral<char> {
    enum { value = 1 };
};
// ... 后面还有其他整数类型
// ... 非整数类型没有相关特化实现
```

和 `enable_if` 一样，只有 _Tp 为整数类型（也就是以上实现了 `_is_integral` 特化的结构）才会有相应的结构定义，非整数类型会因为没有相应结构的定义而出错。

实际上 `std::is_intergral` 这个结构的定义并非如此简单，不同平台的实现也不一样，还经过了一系列的 typedef、派生等处理，不过归根到底最后的思想就是利用偏特化来指定类型实现。

## Constraints

以上就是 C++20 之前对模板的类型萃取，但随着时代的进步，C++20 推出了 concept 和 requires 这两个关键字。

cppreference 上的例子：

```cpp
#include <string>
#include <cstddef>
#include <concepts>
using namespace std::literals;
 
// 概念 "Hashable" 的声明，可被符合以下条件的任何类型 T 满足：
// 对于 T 类型的值 a，表达式 std::hash<T>{}(a) 可编译且其结果可转换为 std::size_t
template<typename T>
concept Hashable = requires(T a) {
    { std::hash<T>{}(a) } -> std::convertible_to<std::size_t>;
};
 
struct meow {};
 
template<Hashable T>
void f(T); // 受约束的 C++20 函数模板
 
// 应用相同约束的另一种方式：
// template<typename T>
//    requires Hashable<T>
// void f(T); 
// 
// template<typename T>
// void f(T) requires Hashable<T>; 
 
int main() {
  f("abc"s); // OK，std::string 满足 Hashable
  f(meow{}); // 错误：meow 不满足 Hashable
}
```

这语法我怎么感觉在 Rust 里见过？（

更多内容：https://en.cppreference.com/w/cpp/language/constraints

