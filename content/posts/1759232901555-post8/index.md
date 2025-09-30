---
date: '2025-08-30T15:39:52+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Dependency Inversion Principle"
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
title: "OODP4 依赖倒置原则(DIP)"
categories: "技术相关"
tags: ["OODP","C#"]
---

# 设计代码结构时，高层模块不应该依赖低层模块，二者都应该依赖其抽象。抽象不应该依赖于细节，细节应该依赖于抽象。

## 错误做法

我们看一个简单的例子，设计一个关系存储的模块，共有三种关系(Relationship)，父，子，同辈。

于是我们给出Relationship枚举和Person类
```C#
public enum Relationship
{
    Parent,
    Child,
    Sibling
}

public class Person
{
    public string Name;
}
```

开始设计低层模块，记录关系的Relationship类，我们随意的使用tuple来记录各种关系，并简单给出添加关系的api，同时将私有的relations成员通过公有的Relations函数暴露出去

```C#

public class Relationships
{
    private List<(Person, Relationship,Person)> relations = [];

    public void AddParentAndChild(Person parent, Person child)
    {
        relations.Add((parent, Relationship.Parent, child));
        relations.Add((child, Relationship.Child, parent));
    }

    public List<(Person, Relationship, Person)> Relations => relations;
}
```

于是我们在高层模块中使用直接使用底层模块，即在Demo构造函数中直接接受Relationships作为参数，当我们想要查询John的孩子是谁时，我们遍历通过Relations得到的关系元组进行查询输出即可。

```C#

public class Demo
{
    public Demo(Relationships relationships)
    {
        var relations = relationships.Relations;
        foreach (var r in relations.Where(x => x.Item1.Name == "John" &&
                    x.Item2 == Relationship.Parent))
        {
            Console.WriteLine($"John has a child {r.Item3.Name}");
        }
    }

    static void Main(string[] args)
    {
        var parent = new Person(){Name = "John"};
        var child1 = new Person(){Name = "Jane"};
        var child2 = new Person(){Name = "Jonny"};
        var relations = new Relationships();
        relations.AddParentAndChild(parent, child1);
        relations.AddParentAndChild(parent, child2);
        new Demo(relations);
    }
} 

```

## 反思环节

注意到，当我们这样做时，由于高层直接依赖于低层，低层是“不可拆换的”。

例如，如果我们不想再使用元组列表作为我们的数据结构，而是想要切换成元组数组，或者直接换成树结构。我们的代码就必须大量重写，不仅要完全重写低层记录关系的类，还得将上层Demo类里直接调用低层类的部分全部重写。

如果这是一个很大的项目架构，这个低层模块可能被很多个高层模块调用，这将带来巨大的维护代价。

从DIP中总结，即是高层模块直接依赖低层模块(Demo类直接使用Relationships类)，抽象依赖于细节(“从关系中查找特定人的孩子”的抽象依赖于了元组和遍历这样的实现细节)

## 正确操作

高层模块不应依赖于低层模块，二者都应该依赖于抽象。那么我们需要一个抽象放在Demo和Relationships之间。

于是我们给出接口IRelationshipBrowser

```C#
public interface IRelationshipBrowser
{
    IEnumerable<Person> FindAllChildOf(string person);
}
```

这个接口即为抽象的一种形式，它代表着“关系浏览器”的抽象概念，而不涉及任何它的实现，只说明它应该有找到某人所有孩子的功能。

于是我们现在可以让高层依赖抽象


这里参数的类型的正是抽象，而不是底层模块本身。与此同时你可以看到API比原来好看的多。

```C#
public Demo(IRelationshipBrowser browser) 
{
    foreach (var r in browser.FindAllChildOf("John"))
    {
        Console.WriteLine($"John has a child {r.Name}");
    }
    
}
```

低层模块的实现也依赖接口的抽象，即低层模块必须实现该接口的功能。

*//*/
public class Relationships : IRelationshipBrowser
{
    private List<(Person, Relationship,Person)> relations = [];

    public void AddParentAndChild(Person parent, Person child)
    {
        relations.Add((parent, Relationship.Parent, child));
        relations.Add((child, Relationship.Child, parent));
    }

    public IEnumerable<Person> FindAllChildOf(string person)
    {
        var relations = Relations;
        return relations.Where(x => x.Item1.Name ==person && x.Item2 == Relationship.Parent).Select(x => x.Item3);
        // low level
    }

    public List<(Person, Relationship, Person)> Relations => relations;
}
```

这里函数的实现使用了LINQ来简化。

注意到此时相比于之前的代码，当我们想要换一种Relationships，如用树來实现同样的功能时，我们不再需要更改上层模块了。此时只需要写一个类，比如说TreeRelationships，让它实现IRelationshipBrowser接口，在所有上层调用中就可以直接使用TreeRelationships类了（因为上层模块接受所有实现了IRelationshipBrowser的类作为参数）。从直观理解上，上层模块只关心下层提供什么样的服务，所以应只依赖于抽象，这样下层即使更改了服务的提供商，只要能提供目标服务，也能正确地运行。

