---
title: CRTP 在多继承中的妙用
date: 2026-01-29 15:50:46
tags:
  - C++
  - 语言特性
categories:	
  - 编程开发
---

多态是 C++ 面向对象编程的一个重要并且非常常用的特性，我们可能基于 `Base` 类实现了 `Derived1` 和 `Derived2` 类。根据开发的需求，我们可能继续分别派生 `Derived1` 和 `Derived2` 到 `Derived1_A` 和 `Derived2_A` 类。
如果此时我们有了新的接口需求，但不能修改 `Base` 类（比如我们的开发准则不允许我们改动这部分代码，况且 `Derived1` 和 `Derived2` 也不需要这个接口），因此只能新增一个 `Interface` 接口类来为 `Derived1_A` 和 `Derived2_A` 提供新的接口。此时问题就出现了，如果 `Interface` 的实现中，我们可能需要访问到 `Base` 类中的成员变量，那么如何实现呢？最直观的想法是为 `Derived1_A` 和 `Derived2_A` 分别实现 `Interface`，但是有很多实现可以共用，我们没必要写重复的代码，但是 `Interface` 类本身又无法访问到 `Base` 类的成员（不能让 `Interface` 继承 `Base`，这样 `Derived1_A` 与 `Derived2_A` 多继承时就存在问题），该怎么办呢？此时 CRTP 就可以发挥它的妙用了。

<!-- more -->

CRTP (Curiously recurring template pattern, 奇异递归模板模式) 是 C++ 中的一种常用的元编程设计范式，其最常被提及的用途就是“静态多态”，减少动态转换的开销，其他基本范式如：

```cpp
template <typename Derived>
class Base
{
public:
    const Derived * asDerived() const { return static_cast<const Derived *>(this); }
    Derived * asDerived() { return static_cast<Derived *>(this); }
};

class Derived : public Base<Derived> {};
```

CRTP 的用法当然不止如此，让我们来看看他是如何解决开头的问题。

```cpp
class Base 
{
public:
    void foo();
};
class Derived1 : public Base {};
class Derived2 : public Base {};

class Interface 
{
public: 
    virtual void accessBase() = 0;
};
class Derived1_A : public Derived1, public Interface {};
class Derived2_A : public Derived2, public Interface {};
```

在目前上方这种写法下，我们无法提取一个公共代码来实现 `accessBase` 方法来访问基类 `Base` 中的成员。现在，我们就可以施展 CRTP 的魔法来实现我们想要的效果了。现在我们引入 CRTP 类 `InterfaceHelper`。

```cpp
template <typename Derived>
class InterfaceHelper : public Interface
{
public:
    void accessBase() override
    {
        asBase()->foo();
    }
private:
    const Derived * asDerived() const { return static_cast<const Derived *>(this); }
    Derived * asDerived() { return static_cast<Derived *>(this); }

    /// static_cast 只能在有继承关系的类中转换，所以我们需要先将当前类转换为 Derived，
    /// 并静态保证 Derived 是 Base 的子类
    const Base * asBase() const { return static_cast<const Base *>(asDerived()); }
    Base * asBase() { return static_cast<Base *>(asDerived()); }
};

template class InterfaceHelper<Derived1_A>;
template class InterfaceHelper<Derived2_A>;

class Derived1_A : public Derived1, public InterfaceHelper<Derived1_A> {};
class Derived2_A : public Derived2, public InterfaceHelper<Derived2_A> {};
```

基于这样一个 `InterfaceHelper`，我们完美实现了我们想要的效果，接下来只需要在 `Interface` 中添加接口，在 `InterfaceHelper` 中实现接口的公共逻辑就可以了。C++ 元编程就是这么好使。
