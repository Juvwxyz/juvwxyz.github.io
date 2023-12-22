---
title: 『C++20』简写函数模板
categories:
  - 语言律师
tags:
  - C++
  - 函数模板
  - 简写函数模板
  - 语法糖
date: 2023-11-11 11:11:11
---

<iframe src="//player.bilibili.com/player.html?bvid=BV1KC4y1S7gX&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

简写函数模板是C++20引入的一个新特性，是一个非常实用的语法糖。

在简写之前，自然是有完整语法的，让我们先来复习一下声明函数模板的完整语法：

```cpp
template<class T1, class T2>
void func(T1 arg1, T2 arg2) {
    /*function body*/
}
```

要声明一个函数模板，首先是`template`关键字，后随一对尖括号括起来的模板形参列表。接下来的部分就和普通的函数声明语法一样了，并且模板形参列表中引入的名字，可以作为类型名用在函数声明中。

用模板形参来模板化函数形参是最常见的函数模板用法。并且，由于模板实参推导的存在，在使用这些函数模板时，不必显式的写出模板实参。

```cpp
// 推导出 T1 = int，T2 = double
// 实例化并且调用func<int, double>(0, 0.0)
func(0, 0.0);
```

模板实参推导是使用函数模板时的便利特性，为了简化函数模板的声明，C++20引入了简写函数模板的语法：

```cpp
// 虽然看起来是个函数声明，但这实际上是个函数模板声明
void func(auto arg1, auto arg2) {
    /*function body*/
}
```

这个简写的形式与上方的完整语法完全等价。简而言之，如果函数声明的形参列表中出现了`auto`占位符类型，那么这个函数声明会声明一个函数模板，并且每一个`auto`占位符都会为这个函数模板追加一个虚设的模板形参。

这些以`auto`占位符声明的形参同样可以添加各种限定符和修饰符，例如cv限定、引用修饰符、指针修饰符等。

```cpp
//这两个foo的声明等价
void foo(const auto& a1, auto (*a2)[4]);

template<class T1, class T2>
void foo(const T1 &a1, T2 (*a2)[4]);
```

形参包也可以简写：
```cpp
//这两个foo的声明等价
void foo(auto&&...args);

template<class...T>
void foo(T&&...args);
```

总而言之，将常规函数模板声明的函数形参列表中的类型名换成`auto`即可。当然，只有函数形参列表中出现的`auto`有这个效果，函数的类型说明符、尾随返回类型、或者函数体中的`auto`还是原来的意思，即让编译器推导实际类型。

如果需要模板化函数的其他部分，还是要显式地写出模板形参列表。此外，简写函数模板也只对类型模板形参有效，非类型模板实参或者模板模板形参也不能简写。

```cpp
// 模板化返回类型，必须写出模板形参列表
// 已有模板形参列表的情况下依然可以使用简写形式
// 简写的参数会在模板形参列表后面追加形参
template<class R>
R convert_to(auto arg);

// 上述声明与此等价
template<class R, class T>
R convert_to(T arg);

// 模板模板形参、非类型模板形参无法简写
template<template<class, size_t>class SizedContainer, class T, size_t N>
void bar(SizedContainer<T,N>&& container);
```

简写函数模板的`auto`占位符可以受概念制约：

```cpp
#include <concepts>
// 仅接受整数类型
void bar(std::integral auto i);

// 等价于：
template <std::integral T>
void bar(T i);
```

简写函数模板版同样可以特化：

```cpp
template<>
void bar(unsigned long long i) {
    std::cout << "unsigned long long";
}
```

总体而言，这是一个很实用的语法糖，在仅需模板化函数参数列表时能够简化很多代码。实际上这也不能完全称得上新特性，它来自C++14就有的泛型lambda表达式，现在C++20终于将它扩展到了函数。

```cpp
// C++14: 泛型lambda
auto lambda = [](auto arg) {
    std::cout << arg << '\n';
}
```

## 参考

- [cppreference - 简写函数模板](https://zh.cppreference.com/w/cpp/language/function_template#.E7.AE.80.E5.86.99.E5.87.BD.E6.95.B0.E6.A8.A1.E6.9D.BF)
