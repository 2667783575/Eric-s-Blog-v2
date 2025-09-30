---
date: '2025-08-27T10:59:07+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Open-Close Principle"
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
title: "OODP1 开闭原则(OCP)"
categories: "技术相关"
tags: ["OODP","C#"]
---

# 软件实体应对拓展开放，对修改关闭

## 以规格模式为例

### 不好的写法


``` C#
public enum Color
{
    Red,
    Green,
    Blue,
}

public enum Size
{
    Small, Medium, Large, Yuge
}

public class Product
{
    public string Name;
    public Color Color;
    public Size Size;

    public Product(string name, Color color, Size size)
    {
        if (name == null)
        {
            throw new ArgumentNullException(paramName: nameof(name));
        }

        Name = name;
        Color = color;
        Size = size;
        
    }
}
```

我们有一个产品类，上级需要我们使用size过滤产品，现在我们来做产品的过滤类

``` C#

public class ProductFilter
{
    public static IEnumerable<Product> FilterBySize(IEnumerable<Product> products, Size size)
    {
        foreach(var p in products)
            if (p.Size == size)
                yield return p;
        
    }
}
```
注意到，如果上级又需要根据color过滤产品，我们又需要打开ProductFilter类，添加FilterByColor，而如果上级又需要我们同时根据color和size过滤产品，我们还需要再添加FilterByColorAndSize。

如果Product有更多属性，这意味着我们需要反复的修改ProductFilter类，而该类可能已经交付了。因此这是一个非常不好的设计。

### 规格模式

首先我们定义两个接口
```C#
public interface ISpecification<in T>
{
    bool IsSatisfied(T t);
}

public interface IFilter<T>
{
    IEnumerable<T> Filter(IEnumerable<T> items, ISpecification<T> spec);
}
```
可以理解成ISpecification处理t是否满足某个条件，而IFilter返回传入的items中所有满足传入的spec规格的项目。

那么此时Filter只要这样写，之后便可以固定住不再开放修改了。
```C#
public class BetterFilter : IFilter<Product>
{
    public IEnumerable<Product> Filter(IEnumerable<Product> items, ISpecification<Product> spec)
    {
        foreach (var i in items)
        {
            if (spec.IsSatisfied(i))
            {
                yield return i;
            }
        }
    }
}
```

而基础的规格类也很简单

```C#
public class SizeSpecification(Size size) : ISpecification<Product>
{
    public bool IsSatisfied(Product t) => t.Size == size;
    
}
public class ColorSpecification(Color color) : ISpecification<Product>
{
    public bool IsSatisfied(Product t) => t.Color == color;
}

```

如果我们需要规格的组合

```C#
public class AndSpecification<T>(ISpecification<T> first, ISpecification<T> second) : ISpecification<T>
{
    private ISpecification<T> _first = first ?? throw new ArgumentNullException(paramName: nameof(first)), _second = second ?? throw new ArgumentNullException(paramName: nameof(second));


    public bool IsSatisfied(T t)
    {
        return _first.IsSatisfied(t) && _second.IsSatisfied(t);
    }
}
```

只需要用AndSpecification将不同的Specification结合到一起生成新的Specification即可。

注意到开发过程中BetterFilter类完全不需要修改，只需要拓展地多写几个实现ISpecification的规格类就可以完成功能的扩展。