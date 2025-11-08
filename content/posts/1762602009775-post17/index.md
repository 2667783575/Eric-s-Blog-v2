---
title: "reference_wrapper 的模板实现"
date: 2025-11-08
draft: false
description: "本文记载了手动模拟cpp标准库实现reference_wrapper的过程"
tags: ["cpp"]
---

## 什么是reference_wrapper

reference_wrapper就是引用包装器，它是一个类模板，可以从它生成obj的模板类，使得该类产生的对象具有和obj的引用近乎完全一样的行为。

## 为什么要有reference_wrapper

在cpp中，如`std::thread`,`std::bind`这样的类只支持接受值的拷贝，而不接受引用。

### 为什么`std::thread`只接受值拷贝？

#### 1. 多线程资源生命周期很危险, `std::ref`可以强迫程序员知道自己在做什么

- 如果默认使用引用传递(左值引用`&`或万能引用`&&`)，则很容易出现生命周期问题
- 你在某个大括号上下文中传入`std::thread`一个局部变量的引用，当离开此上下文后，线程完全有可能仍在执行，
- 而该变量已经被释放

下面代码假设`std::thread`默认传引用

```cpp

void do_something(int& a){
    std::this_thread::sleep_for(std::chrono::seconds(10));//模拟线程一开始做了一些工作，耗时10s
    std::cout << a << std::endl;
}

void do_and_start_a_thread(){
    int x = 1;
    std::thread t(do_something, x);
    t.detach();
    std::this_thread::sleep_for(std::chrono::seconds(1));//模拟这里做了一些工作，只用了1s
}

int main(){
    do_and_start_a_thread();
}
```

从上面的代码看，`do_and_start_a_thread()`函数启动了线程t并传入局部变量x，我们假设`std::thread`以引用的格式接受了x，
并且将x传递给了t，此时线程t上的`do_something`函数和`do_and_start_a_thread`函数并行，且显然1s后`do_and_start_a_thread`函数先结束
此时x被释放掉了，而又过了9s，当`do_something()`运行到访问a的段，a所引用的x却已经被释放掉了吗，错误就会发生。

上面反映了cpp多线程资源生命周期管理的危险性，所以程序员使用`std::ref`时，就会意识到自己正在进行多线程中的引用，减小多线程问题发生的概率。

#### 2. `std::thread`需要先构造新线程，然后在新线程的上下文中使用参数，这意味着必须把参数先存起来，然而引用是不能存储的。

无需多言

#### 3. 一致性原则

这里也没什么好说的，lambda等默认也是值传递，cpp应保持各个方面的一致性。

### 总结

所以必须要有reference_wrapper这样的类，既可以被作为对象拷贝，又可以作为obj的引用发挥相应的作用。

## 实现reference_wrapper

### reference_wrapper应具备的功能

我们先前说reference_wrapper必须可以作为对象拷贝，同时又可以作为引用发挥作用。

前者没什么好说，默认拷贝构造函数就可以了。而后者，我们要实现引用的指向原对象的作用，最自然的，当然是想到用指针指向原对象，而如何作为引用使用，只要实现类型转换到T&的重载即可。

### 初版代码

```cpp
#include <functional>
#include <memory>
#include <type_traits>
#include <utility>
template <typename T> class reference_wrapper {
private:
    // 存储引用的指针
    T *_ptr;

    static T *_s_fun(T &ref) noexcept {
        return &ref;
    }

public:
    template <typename U>
    explicit reference_wrapper(U &&ref) : _ptr(_s_fun(ref)) {}

    reference_wrapper(const reference_wrapper &other)            = default;
    reference_wrapper &operator=(const reference_wrapper &other) = default;
                       operator T &() const noexcept { return this->get(); }
    T                 &get() const noexcept { return *_ptr; }

};
```

这是一个非常简单，也漏洞百出的模板，在上述代码中，我们实现了构造函数获取指针，存储指针，当需要作为`T&`使用时自动转换给出原对象的类引用功能，并给出了默认的拷贝构造函数。

此时，它已经可以使用了。

```cpp

#include "eee.h"
#include <assert.h>
#include <functional>
#include <iostream>
#include <string>
#include <thread>
#include <vector>
void func(int &x) { x = 200; }

int main() {
    int                    a = 0;
    reference_wrapper<int> a_r(a);
    func(a_r);
    assert(a == 200);
}
```

通过编译

但是它有着很多问题

### 特殊情况下，拷贝构造函数被模板构造函数劫持

```cpp
template <typename U>
explicit reference_wrapper(U &&ref) : _ptr(_s_fun(ref)) {}

reference_wrapper(const reference_wrapper &other)            = default;
```

这段代码中模板构造函数和拷贝构造函数同时存在

考虑下面的情况

```cpp
int s = 0;
reference_wrapper<int> a = s;
reference_wrapper<int> b(a);
```

思考b能否被正确构造出来，即`reference_wrapper(a)`是`reference<int>`还是`reference<reference<int>>`。
一般来说是`reference<int>`的，因为cpp的设计中non-template函数比template的函数优先级高，防止了简单情况下问题的暴露。
然而在复杂情况下仍有可能失败，为了彻底杜绝错误，应当添加匹配，当U是`reference_wrapper<T>`类型时，触发SFINAE，保证被执行的是拷贝构造函数。

在类内添加几个模板

```cpp

    template <typename Tp> using raw_t = typename std::remove_cv_t<std::remove_reference_t<Tp>>;
    //去除cvr
    template <typename _T1, typename _T2 = raw_t<_T1>>
    using not_same                           = typename std::enable_if<!std::is_same_v<reference_wrapper, _T2>>;
    template <typename _T1> using not_same_v = not_same<_T1>::type;

```

然后修改构造函数前的模板判断

```cpp
template <typename U, typename = not_same_v<U>>
    explicit reference_wrapper(U &&ref)
        : _ptr(_s_fun(ref) {}

```

上面代码中，`raw_t<T>`是T去除cvr之后的类型，`not_same_v<T>`则是要求T去除cvr之后不能等于`reference_wrapper<T>`(上面代码中只写了`reference_wrapper`，省略了`<T>`，这是由于cpp的类型注入)，如果相等，就会触发SFINAE，则模板构造函数失效，编译器会寻找拷贝构造函数等其他合适的函数来执行。
这样的模板保证了模板构造函数无论如何不可能劫持拷贝构造函数

### 无法优雅的去除不能转换成`T *`的类型

必须注意到类的模板签名中的类型T和构造函数模板签名中的类型U不是同一个类型

```cpp
template <typename T> class reference_wrapper {
......
public:
    template <typename U, typename = not_same_v<U>, typename = decltype(_s_fun(std::declval<U>()))>
    explicit reference_wrapper(U &&ref) noexcept(noexcept(_s_fun(std::declval<U>())))
        : _ptr(_s_fun(
              std::forward<U>(ref)) /*完美转发防止使得左右值对号入座进入不同_s_fun函数，这是右值被拒绝的关键条件*/) {}
......
};
```

当传入的类型错误时，比如说

```cpp
double s = 0;
reference<char> r(s);
```

编译器会报错，但这种错误是类型U参与了重载决议，然后编译器发现无法编译，报出硬错误

这是不符合设计哲学的，更好的办法应该是通过SFINAE，在一开始就告诉编译器，这个U不能参与重载，它不符合要求，所以你去找别的函数吧。最后编译器由于找不到别的函数报出软错误，这就要更好一些。

于是做出如下改进

```cpp
    template <typename U, typename = not_same_v<U>, typename = decltype(_s_fun(std::declval<U>()))>
    explicit reference_wrapper(U &&ref): _ptr(_s_fun(ref)) {
```

`std::declval`代表U类型的一个值，然后`_s_fun()`象征接受这个值，测试是否能成功通过该函数，如果可以才允许U参与重载，否则SFINAE

### no_except的传递

我们默认构造函数都是有exception的，然而对于一些类型，构造函数是不存在exception的，这种情况下应将构造函数设为no_except，帮助编译器优化

```cpp
template <typename U, typename = not_same_v<U>, typename = decltype(_s_fun(std::declval<U>()))>
    explicit reference_wrapper(U &&ref) noexcept(noexcept(_s_fun(std::declval<U>()))) : _ptr(_s_fun(ref)) {}

```

内部的no_except测试`_s_fun()`函数对相应U的值是否no_except，如果是则外部的no_except被设为true。

### 防止传入右值

保存右值的引用是危险的，所以我们要防止保存右值的引用

```cpp
    static T *_s_fun(T &&) = delete;

    template <typename U, typename = not_same_v<U>, typename = decltype(_s_fun(std::declval<U>()))>
    explicit reference_wrapper(U &&ref) noexcept(noexcept(_s_fun(std::declval<U>())))
        : _ptr(_s_fun(
              std::forward<U>(ref)) /*完美转发防止使得左右值对号入座进入不同_s_fun函数，这是右值被拒绝的关键条件*/) {}
```

我们做了两件事，一是删除`_s_fun`的右值引用版本，二是将构造函数中给`_ptr`赋值处将传给`_s_fun()`的ref进行完美转发。
完美转发使得传给`_s_fun`的右值仍是右值，保证右值会被拒绝，触发SFINAE

### 使用安全的取址，防止重载`&`导致的错误

考虑以下代码，由于`&`运算符被重载，我们本来的取地址操作恒定被给出0作为结果。

```cpp
struct Bad{
    Bad * operator &(){
        return 0;
    }
    int a = 10;
}

int main(){
    Bad bad;
    reference_wrapper<Bad> bad_r(bad);
    std::cout << r->get().a;
}
```

这显然是不对的

修改原代码为

```cpp
    static T *_s_fun(T &ref) noexcept {
        return std::addressof(ref);
        // 使用std::addressof防止重载operator&导致的问题
    }
    // 删除右值引用版本，防止传入临时对象
    static T *_s_fun(T &&) = delete;
```

### 针对可调用对象的调用处理

注意到可调用对象也可以被取引用，并且他们的引用也可以被调用，因此reference_wrapper需要重载()运算符，以支持调用功能

```cpp
template <typename... Args>
typename std::__invoke_result<T &, Args...>::type operator()(Args &&...args) const
    noexcept(std::__is_nothrow_invocable<T &, Args...>::value) {
    return std::__invoke(get(), std::forward(args)...);
}
```

### 最后给出方便的工厂

```cpp
template <typename T> reference_wrapper(T &) -> reference_wrapper<T>;
template <typename T> using raw_T = typename std::remove_cv_t<std::remove_reference_t<T>>;
template <typename T> inline reference_wrapper<T> my_ref(T &i) noexcept { return reference_wrapper<raw_T<T>>(i); }
template <typename T> inline reference_wrapper<T> my_cref(const T &i) noexcept {
    return reference_wrapper<const raw_T<T>>(i);
}
template <typename T> void                              my_ref(const T &&)  = delete;
template <typename T> void                              my_cref(const T &&) = delete;
template <typename T> inline reference_wrapper<T>       my_ref(reference_wrapper<T> _t) noexcept { return _t; }
template <typename T> inline const reference_wrapper<T> my_cref(reference_wrapper<T> _t) noexcept { return {_t.get()}; }
```

最后两个工厂重载是为了防止wrapper包裹wrapper

### 完成体

```cpp
#include <functional>
#include <memory>
#include <type_traits>
#include <utility>
template <typename T> class reference_wrapper {
    // 去除引用和const volatile修饰
    template <typename Tp> using raw_t = typename std::remove_cv_t<std::remove_reference_t<Tp>>;
    // SFINAE判断是否传入的T与reference_wrapper相同类型
    template <typename _T1, typename _T2 = raw_t<_T1>>
    using not_same                           = typename std::enable_if<!std::is_same_v<reference_wrapper, _T2>>;
    template <typename _T1> using not_same_v = not_same<_T1>::type;

private:
    // 存储引用的指针
    T *_ptr;

    static T *_s_fun(T &ref) noexcept {
        return std::addressof(ref);
        // 使用std::addressof防止重载operator&导致的问题
    }
    // 删除右值引用版本，防止传入临时对象
    static T *_s_fun(T &&) = delete;

public:
    // not_same_v用于当传入的U类型与reference_wrapper相同时，触发SFINAE，防止普通构造函数被触发
    // 使得可以正常调用拷贝构造函数
    // decltype用于检测_s_fun能否被调用(是否允许U取地址被转化为T*)，若不能则触发SFINAE失败
    template <typename U, typename = not_same_v<U>, typename = decltype(_s_fun(std::declval<U>()))>
    explicit reference_wrapper(U &&ref) noexcept(noexcept(_s_fun(std::declval<U>())))
        : _ptr(_s_fun(
              std::forward<U>(ref)) /*完美转发防止使得左右值对号入座进入不同_s_fun函数，这是右值被拒绝的关键条件*/) {}

    reference_wrapper(const reference_wrapper &other)            = default;
    reference_wrapper &operator=(const reference_wrapper &other) = default;
                       operator T &() const noexcept { return this->get(); }
    T                 &get() const noexcept { return *_ptr; }

    // 针对函数调用运算符的重载，使可调用对象的引用包装器仍可以被调用
    template <typename... Args>
    typename std::__invoke_result<T &, Args...>::type operator()(Args &&...args) const
        noexcept(std::__is_nothrow_invocable<T &, Args...>::value) {
        return std::__invoke(get(), std::forward(args)...);
    }
};
template <typename T> reference_wrapper(T &) -> reference_wrapper<T>;
template <typename T> using raw_T = typename std::remove_cv_t<std::remove_reference_t<T>>;
template <typename T> inline reference_wrapper<T> my_ref(T &i) noexcept { return reference_wrapper<raw_T<T>>(i); }
template <typename T> inline reference_wrapper<T> my_cref(const T &i) noexcept {
    return reference_wrapper<const raw_T<T>>(i);
}
template <typename T> void                              my_ref(const T &&)  = delete;
template <typename T> void                              my_cref(const T &&) = delete;
template <typename T> inline reference_wrapper<T>       my_ref(reference_wrapper<T> _t) noexcept { return _t; }
template <typename T> inline const reference_wrapper<T> my_cref(reference_wrapper<T> _t) noexcept { return {_t.get()}; }
```
