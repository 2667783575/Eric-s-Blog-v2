---
date: '2025-08-20T18:25:46+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "读书有感"
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
title: "关于cpp中const返回值"
categories: "技术相关"
tags: ["cpp"]
---

## 背景

两个月前看了一点Effective Modern Cpp，后来听说作者Scott Mayer写了三本书，Cpp三部曲(E C, More E C, E Modern C)。于是搁置了Effective Modern Cpp，转而去看Effective Cpp。当时看到作者的一个观点是函数返回值优先const
即优先写
``` cpp
const std::string func()...
```
而不是
``` cpp 
std::string func()...
```
原因是有人会在if语句里做判断，发生typo，将判断写成赋值
如
``` cpp
Matrix64 a, b, c;
...
if (a * b = c){ // should be "=="

}
```
若*的重载函数返回const Matrix64，编译器就会报错，否则就难以找到错误发生的地方。
听起来非常有道理，于是书上说
> 一个函数返回一个 constant value（常量值），常常可以在不放弃安全和效率的前提下尽可能减少客户的错误造成的影响。

然而今天机缘巧合之下我去问dicksip我们是不是应该在cpp的函数返回值上默认const，它却给出了否定的答案。
其实原因也非常简单，只是我之前没有去想

## 为什么我们不应该广泛使用const返回值?

### 原因1 降低性能(阻止移动语义,RVO,NRVO)
在现代cpp中，函数返回值const会影响移动语义，移动语义需要非常量右值
在没有const的情况下
``` cpp
Data getData(){
  Data data;
  ...
  return data;
}
Data receiver = getData();// 触发移动语义，甚至RVO,NRVO

// 如果 
// const Data getData();
// 则只能触发拷贝。
```

### 原因2 阻碍现代范式
如
#### 链式调用(Fluent Interface)
``` cpp
class MyString{
public:
  MyString& toUpper();
}


proccessString.toUpper();//没有const时才能写
```
#### 就地操作

``` cpp
void modifyValue(std::string&& str);


modifyValue(getValue()) //高效接受右值处理
```

#### 一种惯用法

``` cpp
// 从流中读取数据并检查是否成功
if (std::cin >> value) { ... }

// 在循环中获取锁并检查是否成功
while (std::unique_lock lock(mutex, std::try_to_lock)) { ... }

// 获取一个资源指针并检查是否有效
if (auto ptr = acquireResource()) { ... }

```
这里unique_ptr和unique_lock等都必须接受非常量右值，如果const就无法构造。

### 原因3 更好的现代方案

原先这种写法解决的问题在今天已经不是问题，现代编译器都可以对if(),while()等中出现的异常赋值操作发出Warning
我们没有必要为了有限的好处牺牲掉众多现代cpp的优秀特性，那显然是因噎废食。

### 原因4 一致性原则

cpp标准库没有那样做，我们应该与cpp标准库保持一致

## 什么时候应该用const返回值?

返回引用和指针时，如果不希望被修改内部数据，**必须**使用const返回值，
此处较为简单，省略

## 总结

我不想说什么“我们应该在读书的时候多加思考”之类的废话。很多时候，我们就是没有办法想到那些东西，才会陷入歧途。所以与其说我们要多思考，不如讲我们可以从哪些方面思考。

### 1. 考虑书的时代性

*Effective Cpp*这本书是Scott Mayer三部曲中的第一本，初版于1991年。这个时间就注定这本书很难完全适用今天的C++23(甚至2c)时代，时过境迁，我们在计算机这样的火热领域，看到如此之大的年代差别时，就应该对书的内容保持较大的警惕了。

### 2. 考虑自己的经验

虽然很多时候很多学科确实会冲击我们的经验，如离散数学，概率论，实分析中很多内容(比如0-1间有理数的测度为0)都和经验相悖，这些实实在在地让我们这样的工科大学生渐渐放弃了我们的经验。然而当理论和经验发生冲突时，进行一个彻底的探究，来搞清楚孰对孰错也是件很要紧的事。如果是经验错了，比起放弃经验和直觉，我们更重要的是培养正确的新的经验和直觉，对我们直觉所依赖的东西进行一个彻底的修复，将错的感觉清楚，正确的理论形成直觉，这才是求实的态度。

当我们看到返回值应该const这样激进的言论的时候，我们要去质疑，毕竟值优先const是很常见的说法，而返回值const则很少见了。在我们平时阅读的开源代码中，将返回值const的写法也是极其罕见的。种种迹象表明，这一个item是极其有悖于常识的。

### 3. 实践出真知

在足够多，足够严谨的工程实践中，我们会找到正确的答案，一切不合适的理论自然会不攻自破，所以多参与工程一定是好的。