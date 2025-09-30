---
date: '2025-08-31T21:33:39+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Builder Pattern"
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
title: "OODP6 构造器模式 (Builder)"
categories: "技术相关"
tags: ["C#","OODP"]
---

# 提供一个简明的 API 来逐步构建一个复杂对象，将其构建过程与它的表示分离开。


## 引入

先来看一段C#自带的Builder模式的应用

StringBuilder

假设我们想要构建Html元素
```C#
static void Main(string[] args)
{
    var hello = "hello";
    var sb = new StringBuilder();
    sb.Append("<p>");
    sb.Append(hello);
    sb.Append("</p>");
    Console.WriteLine(sb.ToString());
    sb.Clear();
    var words = new[] { "hello", "world" };
    sb.Append("<ul>");
    foreach (var word in words)
    {
        sb.AppendFormat("<li>{0}<li>", word);
    }
    sb.Append("</ul>"); 
    Console.WriteLine(sb.ToString());
}
```


我们现在想要写一个HtmlElement类将Html元素封装起来，它需要能够包含标签名，文本和子元素。
```C#
public class HtmlElement
{
    public string Name, Text;
    public List<HtmlElement> Elements = []; 

    public HtmlElement()
    {
        
    }

    public HtmlElement(string name, string text)
    {
        Name = name ?? throw new ArgumentNullException(nameof(name));
        Text = text ?? throw new ArgumentNullException(nameof(text)); 
    }

    
}

```

然后我们希望HtmlElement可以优雅的转换为字符串（应包含缩进）

```C#
    // 在 HtmlElement 类中
    private const int IndentSize = 2; // 定义缩进为两个空格


    private string ToStringImpl(int indent)
    {
        var sb = new StringBuilder();
        var i = new string(' ', indent * IndentSize);
        sb.AppendLine($"{i}<{Name}>");
        if (!string.IsNullOrEmpty(Text))
        {
            sb.Append(new string(' ', (indent+1) * IndentSize));
            sb.AppendLine(Text);
        }

        foreach (var e in Elements)
        {
            string s = e.ToStringImpl(indent + 1);
            sb.Append(s);
        }
        sb.AppendLine($"{i}</{Name}>");
        return sb.ToString();
    } //使用递归DFS实现了元素的字符串输出

    public override string ToString()
    {
        return ToStringImpl(0);
    }

```


我们仍然不满意，还想要一个更简单的构建方式，于是我们自己写一个HtmlBuilder类

## HtmlBuilder

考虑HtmlBuilder应该以一个根元素为基础，可以添加子元素，可以清空子元素

```C#
public class HtmlBuilder
{
    private readonly string rootName; 
    private HtmlElement _root = new HtmlElement();
    
    public HtmlBuilder(string rootName)
    {
        this.rootName = rootName ?? throw new ArgumentNullException(nameof(rootName));
        _root.Name = rootName ?? throw new ArgumentNullException(nameof(rootName));
    }

    public void AddChild(string childName ,string childText)
    {
        var e = new HtmlElement(childName, childText);
        _root.Elements.Add(e);
    }

    public override string ToString()
    {
        return _root.ToString();
    }

    public void Clear()
    {
        _root = new HtmlElement{Name = rootName};
    }
}
```

上述代码中专门保留rootName是因为我们希望Clear()后仍然保持其根元素

调用
```C#
public class Demo
{
    static void Main(string[] args)
    {
        var builder = new HtmlBuilder("ul");
        builder.AddChild("li","hello");
        builder.AddChild("li","world");
        Console.WriteLine(builder.ToString());
    }
}
```

我们就得到了较为美观的输出

```html

<ul>
  <li>
    hello
  </li>
  <li>
    world
  </li>
</ul>

```

## 流构建器

注意到StringBuilder可以使用一种Fluent Interface Pattern的流调用方法
如

```C#
sb.Append("a").Append("b").Append("c");
```

我们希望我们的HtmlBuilder也具有这样的功能，只需要

```C#
public HtmlBuilder AddChild(string childName ,string childText)
{
    var e = new HtmlElement(childName, childText);
    _root.Elements.Add(e);
    return this;
}//将原来的void返回值改为返回对自身的引用即可
```

于是我们可以这样调用

```C#
builder.AddChild("li","hello").AddChild("li","world");
```


## 流式构建器的继承


### 问题

对普通构建者的继承不会引起什么问题，而当使用流式构建者时，问题则棘手起来。

先来看一个简单的例子，我们有Person类，它具有名字和职位两个属性。同时有一个流构建者只处理名字。
```C#
public class Person
{
    public string Name;
    public string Position;
    public override string ToString()
    {
        return $"{nameof(Name)}: {Name}, {nameof(Position)}: {Position}";
    }
}
public class PersonInfoBuilder
{
    protected Person person = new Person();
    //这里是protected因为我们一会将处理继承关系

    public PersonInfoBuilder Called(string name)
    {
        person.Name = name;
        return this;
    }
}
```
现在我们有新的业务需求，即现在需要Builder能同时处理职位的构建，我们遵循开闭原则使用新类继承PersonInfoBuilder

```C#
public class PersonJobBuilder: PersonInfoBuilder
{
    public PersonJobBuilder WorksAsA(string position)
    {
        person.Position = position;
        return this;
    }
}
```

现在我们尝试调用PersonJobBuilder

```C#
// bad code
PersonJobBuilder jb = new PersonJobBuilder();
jb.Called("Li Bad").WorksAsA("Manager");
```

注意到这段代码是报错的，原因是Called方法返回的是一个PersonInfoBuilder，而PersonInfoBuilder不能处理职位相关事项，不具备WorksAsA方法

问题本质：
当 Fluent 方法需要返回 this 时，在继承链中，父类的方法如果返回的是父类类型 (PersonInfoBuilder)，那么子类的方法链就会在调用父类方法后中断，无法继续调用子类的方法。

### 解决方案

使用递归泛型

首先我们来创建一个PersonBuilder的抽象类，将Person的存储和Build方法提取到抽象类里。
```C#
public abstract class PersonBuilder
{
    protected Person person = new Person();

    public Person Build()
    {
        return person;
    }
}
```

然后用递归泛型改写PersonInfoBuilder
```C#
public class PersonInfoBuilder<TSelf> : PersonBuilder
where TSelf : PersonInfoBuilder<TSelf>
{

    public TSelf Called(string name)
    {
        person.Name = name;
        return (TSelf)this;
    }
}
```
理解： 给PersonInfoBuilder添加了一个泛型TSelf，TSelf用来存放子类的类型，于是Called方法就可以通过类型转换return子类的类型，为了保证TSelf一定是子类的类型，我们用where语句限制TSelf继承于该类。

我们可能下意识这样使用
```C#
// bad code
public class PersonJobBuilder: PersonInfoBuilder<PersonJobBuilder>
{
    public PersonJobBuilder WorksAsA(string position)
    {
        person.Position = position;
        return this;
    }
}
```

但是假想如果有类又继承PersonJobBuilder的话，由于PersonJobBuilder被固定，继承PersonJobBuilder的类又将不能正确工作。因此这决不是一个好注意。

正确的写法应该是在PersonJobBuilder上继续泛型

```C#
public class PersonJobBuilder<TSelf> : PersonInfoBuilder<PersonJobBuilder<TSelf>>
    where TSelf : PersonJobBuilder<TSelf>
{
    public TSelf WorksAsA(string position)
    {
        person.Position = position;
        return (TSelf)this;
    }
}
```
理解： PersonJobBuilder也具有类型TSelf的泛型，也用where限制TSelf继承于该类，同时PersonJobBuilder中的泛型从PersonJobBuilder改为PersonJobBuilder<TSelf>，仅此而已

当我们高兴的尝试使用PersonJobBuilder时，我们发现PersonJobBuilder并不能直接构建。这样的泛型类都不能直接使用，必须用一个类继承泛型类，才能使用。

所以我们在Person中写一个Builder类，它继承于PersonJobBuilder<Person.Builder>，同时给Person类添加相应的构建方法

```C#
// 在Person类内
public class Builder:PersonJobBuilder<Builder>
{
}

public static Builder New => new Builder();
```

于是我们现在可以调用

```C#
var person = Person.New.
    Called("Wang").
    WorksAsA("LaoBan").
    Build();
Console.WriteLine(person);
```
得到输出
```output
Name: Wang, Position: LaoBan
```

## 分步构建器

设想这样一个场景，我们现在要做车的创建者，车子具有类型，小轿车(Sedan)或跨界车(CrossOver)，小轿车的尺寸必须在15到17之间，而跨界车必须在17-20之间，如果超出范围就要报错。

我们注意到该构建过程具有明显的分步性，必须先知道类型，才能判断尺寸，这就引出了分步构建者

先定义车类
```C#
public enum CarType
{
    Sedan,
    Crossover
}

public class Car
{
    public CarType type;
    public int WheelSize;
}
```

应用接口隔离原则我们将车的构建过程分成几个独立的接口
```C#
public interface ISpecifyCarType
{
    ISpecifyWheelSize OfType(CarType type);
}
public interface ISpecifyWheelSize
{
    IBuildCar WithWheels(int size);
}
    public interface IBuildCar
{
    public Car Build();
}   
```

其中ISpecifyCarType用来指定车的类型，它随后返回一个ISpecifyWheelSize接口进一步指定车的轮子尺寸，再返回IBuildCar进一步构建车对象

在CarBuilder类中，我们将构建过程的实现封装到Impl里

```C#
public class CarBuilder
{
    private class Impl:ISpecifyWheelSize,ISpecifyCarType,IBuildCar
    {
        private Car _car = new Car();
        public ISpecifyWheelSize OfType(CarType type)
        {
            _car.type = type;
            // work as ISpecifyCarType
            return this;
            // this is ISpecifyWheelSize
        }
        public IBuildCar WithWheels(int size)
        {
            switch (_car.type)
            {
                case CarType.Crossover when size is < 17 or > 20:
                case CarType.Sedan when size is < 15 or > 17:
                    throw new ArgumentException("Car size out of range");
                default:
                    break;
            }
            _car.WheelSize = size;
            // work as ISpecifyWheelSize
            return this;
            // this is IBuildCar
        }


        public Car Build()
        {
            return _car;
        }
    }   
    public static ISpecifyCarType Create()
    {
        return new Impl();
    }
}
```

注意到上述Impl类同时实现了ISpecifyWheelSize,ISpecifyCarType,IBuildCar三个接口，所以它只需要不停return this就可以让构建过程不断递进。

使用如下
```C#
var car = CarBuilder.Create().OfType(CarType.Sedan).WithWheels(20).Build();
```


## 函数式构建器

我们可以使用函数式的方式来构造一个PersonBuilder类，我们让它存储下我们想要对对象进行的所有操作的列表，然后在Build时再对初始对象进行所有这些操作并返回

```C#
public sealed class PersonBuilder
{
    private List<Func<Person, Person>> actions =  new List<Func<Person, Person>>();

    private PersonBuilder AddAction(Action<Person> action)
    {
        actions.Add(p=>
        {
            action(p);
            return p;
        });
        return this;
    }

    public Person Build() => actions.Aggregate(new Person(), (acc, action) => action(acc)); 
    public PersonBuilder Called(string name) => Do(p => p.Name = name);
    public PersonBuilder Do(Action<Person> action)=>AddAction(action);
    
}
```
我们使用sealed关键字密封该类，强调我们不需要继承就能实现想要的功能。

那么我们如何不通过继承来添加新的功能呢？

答案是通过C#的静态拓展方法

```C#
public static class PersonBuilderExtensions
{
    public static PersonBuilder WorksAsA(this PersonBuilder builder, string position)
        => builder.Do(p => p.Position = position);
}
```

此时我们可以直接使用WorksAsA

```C#
var person = new PersonBuilder().Called("Wang").WorksAsA("LaoBan").Build();
```

注意到，函数式创建者的工作方式其实都是类似的，所以我们直接将函数式构建者写为泛型，然后再特化使用

```C#
public abstract class FunctionBuilder<TSubject, TSelf>
where TSelf : FunctionBuilder<TSubject, TSelf>
where TSubject: new()
{
    private List<Func<TSubject, TSubject>> actions =  new List<Func<TSubject, TSubject>>();
    
    private TSelf AddAction(Action<TSubject> action)
    {
        actions.Add(p=>
        {
            action(p);
            return p;
        });
        return (TSelf)this;
    }
    
    public TSubject Build() => actions.Aggregate(new TSubject(), (acc, action) => action(acc)); 
    public TSelf Do(Action<TSubject> action)=>AddAction(action);
    
}

public sealed class PersonBuilder:FunctionBuilder<Person,PersonBuilder>
{
    public PersonBuilder Called(string name)=> Do(p => p.Name = name);
}

public static class PersonBuilderExtensions
{
    public static PersonBuilder WorksAsA(this PersonBuilder builder, string position)
        => builder.Do(p => p.Position = position);
}
    
```

功能仍然不变

## 分派构建器

我们可以将一个对象的属性分拆成不同的方面，交给不同的构建器负责，最后再封装给一个构建器

### 例子

比如我们有一个Person类，它具有StreetAddress,Postcode,City等居住地相关属性和CompanyName,Position,AnnualIncome等工作相关属性

```C#
public class Person
{
    public string? StreetAddress,Postcode,City;
    public string? CompanyName, Position;
    public int AnnualIncome;
    public override string ToString()
    {
        return $"{nameof(StreetAddress)}: {StreetAddress}, {nameof(Postcode)}: {Postcode}, {nameof(City)}: {City}, {nameof(CompanyName)}: {CompanyName},  {nameof(Position)}: {Position}, {nameof(AnnualIncome)}: {AnnualIncome}";
    }
}
```

然后再写一个PersonBuilder类（该类是一个外壳，并不处理具体构建过程）
```C#
public class PersonBuilder // facade
{
    protected Person Person = new Person();
}
```
然后我们写PersonAddressBuilder和PersonJobBuilder两个类继承于PersonBuilder，分别处理地址和工作相关的属性的构建

```C#
public class PersonAddressBuilder : PersonBuilder
{
    public PersonAddressBuilder(Person person)
    {
        Person = person;
    }

    public PersonAddressBuilder At(string streetAddress)
    {
        Person.StreetAddress = streetAddress;
        return this;
    }

    public PersonAddressBuilder WithPostcode(string postcode)
    {
        Person.Postcode = postcode;
        return this;
    }

    public PersonAddressBuilder In(string city)
    {
        Person.City = city;
        return this;
    }
}
public class PersonJobBuilder:PersonBuilder
{
    public PersonJobBuilder(Person person)
    {
        this.Person = person;
    }

    public PersonJobBuilder At(string companyName)
    {
        Person.CompanyName = companyName;
        return this;
    }

    public PersonJobBuilder AsA(string position)
    {
        Person.Position = position;
        return this;
    }
    public PersonJobBuilder Earning(int amount)
    {
        Person.AnnualIncome = amount;
        return this;
    }
}
```

将PersonJobBuilder和PersonAddressBuilder的对象作为PersonBuilder的公共属性

```C#
public class PersonBuilder // facade
{
    protected Person Person = new Person();
    public PersonJobBuilder Works => new PersonJobBuilder(Person); 
    public PersonAddressBuilder Lives => new PersonAddressBuilder(Person);

    public static implicit operator Person(PersonBuilder b) => b.Person;
}
```

上面代码中的operator重载了PersonBuilder向Person的隐式转换，使得PersonBuilder构建对象使用时更方便

于是我们可以这样使用构建器

```C#
var pb = new PersonBuilder();
Person person = pb.Works.At("Huawei").AsA("Manager").Earning(1000000)
    .Lives.At("Nanluoguxiang").In("Beijing").WithPostcode("666");
```

## 总结

1. 构建器是用来构建对象的独立的组件
2. 你可以通过构造函数生成构建器，也可以通过静态函数返回构建器
3. 如果要让构建器实现流式接口，只需要return this，如果还要实现继承，使用递归泛型
4. 一个对象的不同方面可以由不同的构建器通过一个基类并联起来