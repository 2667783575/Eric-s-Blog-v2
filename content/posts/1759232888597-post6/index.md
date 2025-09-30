---
date: '2025-08-27T16:25:47+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Liskov Substitution Principle"
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
title: "OODP2 里氏替换原则(LSP)"
categories: "技术相关"
tags: ["OODP","C#"]
---

# 父类的对象可以被子类的对象替换，而程序的行为不会发生变化

## 例子

我们有Rectangle类
```C#
public class Rectangle
{
    public int Height{get; set; }
    public int Width { get; set; }

    public Rectangle()
    {
        
    }

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }

    public override string ToString()
    {
        return $"{nameof(Width)}: {Height}, {nameof(Height)}: {Height}";
    }
}
```

基于Rectangle类再写Square类

```C#
public class Square : Rectangle
{
    public new int Width
    {
        set { base.Width = base.Height = value; }
    }

    public new int Height
    {
        set { base.Width = base.Height = value; }   
    }
}
```
注意到我们用new关键字hide了父类的Width和Height

此时Square和Rectangle都可以正常工作

```C#
Rectangle rc = new Rectangle();
rc.Width = 20;
rc.Height = 40;
Console.WriteLine("Area: {0}", Area(rc));
Square sq = new Square();
sq.Width = 4;
Console.WriteLine("Area {0}", Area(sq));
```
可以看到结果正常

而由于正方形是一种长方形，我们可以使用父类型Rectangle来持有子类型Square的引用。

那么**问题发生了**

```C#
Rectangle sq = new Square();
sq.Width = 4;
Console.WriteLine("Area {0}", Area(sq));
```

```
输出: Area 0
```
这就违反了里氏替换原则，当子类的对象替换父类时发生了错误。

## 修改方案

将Rectangle中Width和Height的get,set修改为虚，使父类引用执行子类的函数

```C#
public virtual int Height{get; set; }
public virtual int Width { get; set; }
```

同时将Square中的用new来hide父类型改为用override来重写父类虚函数

```C#
public override int Width
{
    set { base.Width = base.Height = value; }
}

public override int Height
{
    set { base.Width = base.Height = value; }   
}
```

则现在不会出现错误

| 特性     | 里氏替换原则 (LSP)                           | 虚函数/重写 (virtual/override)             |
| :------- | :------------------------------------------- | :----------------------------------------- |
| **本质** | **设计原则** (Principle)                     | **语法机制** (Syntax)                      |
| **目的** | 指导如何正确地建立继承关系，确保行为兼容性。 | 实现**运行时多态**，允许子类定制方法实现。 |
| **关系** | **目标** (What & Why)                        | **主要实现手段** (How)                     |


## C#, C++和Java的虚函数辨析

| 特性                   | C#                         | Java                                  |
| :--------------------- | :------------------------- | :------------------------------------ |
| **声明可重写方法**     | 使用 `virtual` 关键字      | **默认就是“虚”的**，无需关键字        |
| **重写方法**           | 使用 `override` 关键字     | 使用 `@Override` **注解**（最佳实践） |
| **阻止重写**           | 方法**不**标记为 `virtual` | 使用 `final` 关键字                   |
| **隐藏方法（非多态）** | 使用 `new` 关键字          | 无关键字（但不推荐，会产生警告）      |

| 特性         | C#                                                      | C++                                                                |
| :----------- | :------------------------------------------------------ | :----------------------------------------------------------------- |
| **默认行为** | **非虚拟** (方法默认不能被重写)                         | **非虚拟** (方法默认不具有多态性)                                  |
| **实现多态** | 基类 `virtual` + 派生类 `override`                      | 基类 `virtual` (派生类推荐使用 `override`)                         |
| **设计哲学** | **“默认关闭”**：出于安全和性能，需要显式开放重写能力。  | **“不为未使用的东西付出代价”**：需要多态时，显式请求并承担其开销。 |
| **你的做法** | **慎重选择**哪些方法设为`virtual`。只开放需要扩展的点。 | **必须**为所有需要多态的方法加上`virtual`，否则无法按预期工作。    |
