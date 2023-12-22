---
title: 『C++23』显式对象形参
categories:
  - 语言律师
tags:
  - C++
  - 非静态成员函数
  - 显式对象形参
  - 推导this
  - 语法糖
date: 2023-11-19 18:17:16
---

<iframe src="//player.bilibili.com/player.html?bvid=BV11j411W72W&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

# 隐式对象形参

在正式介绍显式对象形参之前，让我们先来复习一下什么叫隐式对象形参。我们知道，在调用类的非静态成员函数时，必须通过对象的成员访问运算符来调用。而在函数内，我们可以通过`this`指针来访问这个调用成员函数时的对象。

```cpp
#include <cstdio>

struct A {
    void foo() {
        printf("Call A::foo() from %p\n", this);
    }
};

int main() {
    A a, b;
    printf("address of a:  %p\n", &a);
    a.foo();
    printf("address of b:  %p\n", &b);
    b.foo();
}
```

这一特性背后的机制就是隐式对象形参。在重载决议的过程中，编译器会在成员函数的形参列表最前面加上一个额外的形参。同时，将本次调用所用的对象作为隐含对象实参放置在实参列表的最前面。而`this`指针就是指向这个隐式对象形参。

```cpp
// 重载决议过程中看到的 A::foo 的函数签名：
void A::foo(A&);
```

而非静态成员函数的cv限定与引用限定会影响隐式对象形参的类型。

```cpp
#include <iostream>
#include <memory>
struct A {
    // void A::foo(A&) 对于无引用限定的非静态成员函数，
    // 隐式对象形参的类型是左值引用，但额外规定其可以绑定到右值。
    void foo() { std::cout << "A::foo(A&)\n"; }

    // void A::bar(A&) 对于有左值引用限定的非静态成员函数，
    // 隐式对象形参的类型是左值引用，并且不能绑定到右值。
    void bar() & { std::cout << "A::bar(A&)\n"; }

    // void A::bar(A&&) 对于有右值引用限定的非静态成员函数，
    // 隐式对象形参的类型是右值引用，只能绑定到右值。
    void bar() && { std::cout << "A::bar(A&&)\n"; }

    // void A::baz(const A&) cv限定同样会影响
    // 隐式对象形参的类型。
    void baz() const& { std::cout << "A::baz(const A&)\n"; }
};
int main() {
    A a;
    a.foo();
    std::move(a).foo(); // OK，特殊规定
    a.bar();            // 调用左值引用重载
    std::move(a).bar(); // 调用右值引用重载
    std::move(a).baz(); // OK，const&可以绑定到右值
}
```

通过上面的例子可以看到，当我们需要对隐式对象形参的类型进行限定时，往往要写好几个重载。而很多时候，这些函数的代码几乎一模一样。于是C++23引入了一个新特性——显式对象形参，允许我们将原本看不见的隐式对象形参显式地写出来。接下来我们就看看显式对象形参能够如何帮助我们简化编码。

# 显式对象形参

首先看到显式对象形参的语法形式：声明成员函数时，在第一个参数前加上`this`关键字，表示该参数是显式对象形参。有显式对象形参的成员函数就称为显式对象成员函数，普通的非静态成员函数相应地称为隐式对象成员函数。

```cpp
struct A {
    void foo(this A& self) {}
};

int main() {
    A a;
    // 调用语法与普通成员函数相同
    // 将a作为第一个实参传递给A::foo
    a.foo();
}
```

显式对象形参有一些限制，构造函数、析构函数、静态成员函数、虚函数不能有显式对象形参。

```cpp
struct A {
    // Error, 构造函数不能有显式对象形参
    A(this const A&);
    // Error, 析构函数不能有显式对象形参
    ~A(this const A&);
    // Error, 静态成员函数不能有显式对象形参
    static void foo(this const A&);
    // Error, 虚函数不能有显式对象形参
    virtual void foo(this const A&);

    // 相当于 void foo() const&
    void foo(this const A& self);
};
```

它们也不能有cv限定或者引用限定。若要对参数类型进行限定，应当限定在显式对象形参上。

```cpp
struct A {
    // 相当于 void foo() &
    void foo(this A& self);
    // 相当于 void foo() &&
    void foo(this A&& self);
    // 相当于 void bar() const &
    void bar(this A const & self);
};
```

也不能在显式对象成员函数体内使用`this`指针。所有的成员访问必须通过其第一个参数进行。

```cpp
struct A {
    void bar() const {}
    void foo(this const A& self) {
        self.bar();  // OK, 通过显式对象形参进行成员访问
        this->bar(); // Error, 不能使用this
        bar();       // Error, 没有隐式this->
    }
};
```

显式对象成员函数与隐式对象成员函数可以重载，只要重载决议能够区分二者的参数类型。

```cpp
struct A {
    void foo(this A& a);

    // OK, 隐式对象形参的类型是 const A&
    void foo() const&;
    // OK, 隐式对象形参的类型是 A&&
    void foo() &&;
    // Error, 隐式对象形参的类型是 A&，与第一个重载冲突
    void foo();
};
```

显式对象形参的传参规则与普通的函数参数一样。如果它没有声明为引用，那么传参时会发生复制。并且，它的类型不必与该类相同，只要能够隐式转换即可。

```cpp
struct A {};
struct B {
    // 自定义转换到A
    operator A() const { return {}; }

    // 以值语义传递，传参时发生复制
    void foo (this B b) {}

    // OK, 传参时先转换到A
    void bar(this A a) {}
};
```

更重要的是，在成员函数模板中使用显式对象形参时，它的类型与值类别和其他参数一样可以进行模板实参推导，这也是这一特性被称为`推导this (deducing this)`的原因。

```cpp
struct A {
    template<class T>
    void foo(this T&& self) { }
};
int main() {
    A a; const A& r = a;
    a.foo();            // T -> A&,       this A & self
    r.foo();            // T -> const A&, this A const & self
    std::move(a).foo(); // T -> A,        this A && self
}
```

最后，指向显式对象成员函数的指针是普通函数指针，而不是指向成员的指针。二者有本质区别。

```cpp
struct A {
    void bar(int) {}
    void foo(this A, int) {}
};

int main() {
  A a;
  // p1的类型是 void(A::*)(int)
  auto p1 = &A::bar;
  (a.*p1)(0);
  // p2的类型是 void(*)(A, int)
  auto p2 = &A::foo;
  p2(a, 0);
}
```

# 使用例

## 1、减少重复成员函数的编码

显式对象形参可以简化需要区分const与非const重载的成员函数。例如下面的例子展示了一个类似STL容器的类。它重载了`operator[]`以访问其元素。区分const与非const重载是很常见的需求：对于const对象，返回其元素的const引用；对于非const对象返回非const引用。使用显式对象形参的自动推导，配合转发引用auto&&甚至能推导出右值引用，对于右值实参可返回其元素的右值引用。这使得自定义容器的`operator[]`与原生数组的下标运算行为一致。数组左值的下标运算是左值，而数组右值的下标运算是亡值。现有STL容器的`operator[]`并未遵循这一语义。

```cpp
struct A {
    int* m_data;
    A() : m_data(new int[5]) {}
    ~A() { delete[] m_data; }

    // 有显式对象形参之前，const与非const重载必须写两遍，虽然它们的函数体几乎一模一样
    // int& operator[](size_t i) & {
    //    return m_data[i];
    //  }
    //  const int& operator[](size_t i) const & {
    //    return m_data[i];
    //  }

    // 如果需要进一步区分左值实参和右值实参，甚至还要写第三个重载
    //  int&& operator[](size_t i) && {
    //    return std::move(m_data[i]);
    //  }

    // const && 的场景比较少见
    //  const int&& operator[](size_t i) const && {
    //    return std::move(m_data[i]);
    //  }

    // 使用显式对象形参，可自动推导实参类型，配合std::forward_like
    // 和decltype(auto)可自动推导返回值的引用限定和cv限定，用一个函数
    // 模板即可处理上述四个重载
    decltype(auto) operator[](this auto&& self, size_t i) {
        return std::forward_like<decltype(self)>(m_data[i]);
    }
};
```

## 2、简化CRTP手法

CRTP（奇异递归模板模式，Curiously recurring template pattern）是一种常用的静态多态手法。标准库的类模板std::enable_shared_from_this就用到了它。推导this能够帮助我们简化CRTP。

```cpp
struct inc_op {
    // 配合简写函数模板，自动推导self的类型
    decltype(auto) operator++(this auto&& self) {
        return self.post_inc();
    }
    auto operator++(this auto&& self, int) {
        auto temp = self;
        ++self;
        return temp;
    }
};

struct A : inc_op  {
    int value;
    A& post_inc() { value++; return *this; }
};

int main() {
    A a;
    // 调用 int& inc_op::operator++ <A&> (this A& self)
    ++a;

    // 调用 int& inc_op::operator++ <A&> (this A& self, int)
    a++;
}
```

## 3、简化递归lambda

显式对象形参也可以简化递归lambda。lambda虽然是一个重载了函数调用运算符的匿名类类型，却无法在其函数体中使用`this`指针。只能是出现在类成员函数的lambda捕获并访问其外围的`this`。

```cpp
#include <iostream>
int main() {
    // lambda虽然是类类型，却无法在其成员函数内通过this
    // 访问隐式对象形参，递归lambda必须传一个额外的参数
    // 这样既不直观，也很低效。
    auto fib1 = [](auto&& self, int n) {
        if(n <= 1)
            return n;
        else
            return self(self, n-1)+self(self, n-2);
    };
    std::cout << fib1(fib1, 8) << '\n';

    // 使用显式对象形参可以方便又直观地递归
    auto fib2 = [](this auto&& self, int n) {
        if(n <= 1)
            return n;
        else
            return self(n-1)+self(n-2);
    };
    std::cout << fib2(8) << '\n';
}
```

## 参考

- [cppreference - 显式对象形参](https://zh.cppreference.com/w/cpp/language/member_functions#.E6.98.BE.E5.BC.8F.E5.AF.B9.E8.B1.A1.E5.BD.A2.E5.8F.82)
