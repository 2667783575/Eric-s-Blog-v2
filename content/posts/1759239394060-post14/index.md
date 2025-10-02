---
title: "从零构建任务管理应用：我的前后端分离初体验"
date: 2025-09-30
draft: false
description: "之前在大一上学期的OODP课程中，曾用 JSP + Struts + Spring + Hibernate 技术栈开发过 Web 应用。那时前后端尚未真正分离，甚至没有明确的“前端”和“后端”概念。而这次，我终于体验了一回现代意义上的前后端分离开发。"
tags: ["example","tag"]
---
## 起因

故事的起因是我的[另一篇博文](https://2667783575.github.io/Eric-s-Blog-v2/posts/1759232853936-post1/)，那篇博文只有构思，而没有考虑实际使用Qt/C++ Postgresql来实现这样一个软件难度是非常大的。

然而刚好最近在看C#，也开始了解了ASP.NET Core Web API，又恰好我学过一点点Postgresql，并且想要了解后端架构，于是我认为我有充分的能力去构建项目的后端。

至于前端部分，我对h5,js,css的基础内容已经了解，在现代前端中基本上依赖vue,axios,tailwind也就能很简单的实现功能了。而恰好我对vue的基础的数据绑定略有认识，同时axios的基础使用非常简单，想必前端开发也不在话下。

当然后面的展开证明我不会的部分还是很多的，在实战过程中也接触到好用的tailwind组件库daisyUi，vue开发状态管理库pinia。后端也见到了数据库Migration等有趣的技术，拓宽了视野。

技术栈总结

- **前端**：Vue 3 (Composition API) + Pinia + Axios + Tailwind CSS + daisyUI
- **后端**：ASP.NET Core Web API + Entity Framework Core
- **数据库**：PostgreSQL
- **工具链**：VS Code / swagger, Rider, Git

## 前端开发过程中的收获

### vue

对vue组件管理有了更深刻的认识。Vue 是一个渐进式 JavaScript 框架，核心优势在于组件化开发与响应式数据绑定，能够高效地构建用户界面。有父子组件的嵌套，就必然遇到子组件父组件通信的问题。vue3中通常父组件通过props或者slots向子组件传递数据，而子组件主要通过emit向父组件传递数据。

在具体的使用过程中遇到比较多的问题是常常会忘记ref对象必须使用value，而不能用对象本身。因为ref 是响应式引用，必须通过 .value 访问/赋值，否则会破坏响应式链。在模板中使用 ref 时，Vue 会自动解包（无需 .value），但在 script setup标签v中必须显式使用。

``` js
const obj = ref(null)
obj.value = "foo"  // 正确，修改响应式数据
obj = "foo"        // 错误，丢失响应式能力
```

虽然这可以说是一个非常没有技术含量的问题，但是作为新手的确会经常地写错。

在使用vue时还遇到一个非常棘手的问题，就是全局状态管理问题。

如上文所说，本身的vue只能通过父组件向子组件传props/slots和子组件向父组件传emit来进行数据的上下流动，然而当你遇到多个组件嵌套

```vue
<!-- App.vue -->
<template>
<Compo1 />
</template>
```

```vue
<!-- Compo1.vue -->
<template>
<Compo2 />
</template>
```

```vue
<!-- Compo2.vue -->
<template>
<!-- ...... -->
<template>
```

我们且不考虑更复杂的情况，单单是跨了一层的嵌套，如果你现在想要Compo2.vue获得并执行App.vue的一个函数(比如说axios向服务器发起通信请求，如果将 API 调用逻辑集中在根组件，会导致深层组件难以复用)，你就不得不通过props将函数一层一层传下去

```vue
<!-- App.vue -->
<template>
<Compo1 :getTasks="getTasks"/>
</template>
```

```vue
<!-- Compo1.vue -->
<template>
<Compo2 :getTasks="getTasks/>
</template>
```

```vue
<!-- Compo2.vue -->
<template>
<button @click="handleTasks"></button>
<!-- ...... -->
<template>
<script setup>
defineProps(['getTasks'])
const handleTasks = async ()=>{
    // 执行传进来的getTasks函数
}
</script>
```

这仅仅是两层嵌套的情况，实际开发中可能有更多层嵌套，各种嵌套间要传递各种函数，这种情况一定是糟糕的，我需要更好的办法解决这种问题，在问了ai之后也的确得到了满意的回答

### pinia

Pinia 是 Vue 官方推荐的状态管理库，替代了 Vuex，利用pinia的defineStore函数就可以定义全局状态，在任何组件中都可以直接通过函数获得相应store而使用全局函数和读写全局状态。在一开始我还迟疑是否要向项目引入pinia，因为担心项目变得更复杂，但是在看过pinia官网后，我的疑虑打消了，pinia的使用非常简单，基本使用看了两页就可以直接上手了，因此我毫不犹豫地引入了pinia。

### js的槽点

js的一大槽点是作为一个前端语言，它很不严谨，可能触发各种各样的问题，让你的页面表现不正常。如果有机会的话以后一定要学学ts

### axios

axios确实是一个非常好用非常简单的框架，

```js
// 定义全局状态
export const useTaskStore = defineStore('tasks', {
  state: () => ({
    tasks: [],
    loading: false,
    error: null,
    selectedTask: null,
  }),//全局变量
  actions: {
    async fetchTasks() {
      this.loading = true
      this.error = null
      try {
        const response = await axios.get('http://localhost:5005/taskitems')
        this.tasks = response.data;
      } catch {
        this.error = "get failed"
        console.error(this.error)
      } finally {
        this.loading = false
      }
    },
    async postTask(task){
        this.loading = true
        this.error = null
        try{
            const response = await axios.post('http://localhost:5005/taskitems',task)
            this.tasks.unshift(response.data)
        }catch{
            this.error = "post failed"
            console.error(this.error)
        }finally{
            this.loading = false
        }
    },
    async updateTask(task){
        this.loading = true
        this.error = null
        try{
            const response = await axios.put('http://localhost:5005/taskitems/'+task.id,task)
            const index =  this.tasks.findIndex(t=>t.id===task.id)
            if(index != -1){
                this.tasks[index] = response.data
                this.selectedTask = response.data
            }
        }
        catch{
            this.error = "put failed"
            console.error(this.error)
        }
        finally{
            this.loading = false;
        }
    }
    ,
    async deleteTask(id){
            this.loading = true
        this.error = null
        try{
            await axios.delete('http://localhost:5005/taskitems/'+id)
            const index = this.tasks.findIndex(t => t.id === id);
            if (index !== -1) {
                this.tasks.splice(index, 1)
            }
        }
        catch{
            this.error = "put failed"
            console.error(this.error)
        }
        finally{
            this.loading = false;
        }
    }
  }// 全局函数
})

```

Note: 这里为了短期方便直接使用了应编码，实际开发更推荐axios.create({ baseURL: ... })，然后单独全局配置baseURL，方便统一管理API地址 

可见只要非常简单的使用axios的函数就可以实现发送请求的基本功能。

### tailwindcss和daisyUI

虽然我在做很多小组任务时对页面外观都几乎没有要求，只要实现功能就行。但是自己做的东西，还是希望做的让自己满意，所以在询问ai之后，ai推荐使用tailwindcss和daisyUI进行外观配置

首先tailwindcss是一个主打utility式的css的工具，它主要的优越之处在于抛弃了原本的先写各种class使用的思路，而是使用写好的utility，独立而分散的配置每一个标签的外观。而daisyUI是一个基于tailwindcss的组件库，它预先定义了很多组件的外观和配置，如卡片，列表等可以直接使用，并且daisyUI 提供了主题切换能力，适合快速构建美观界面。总的来说使用tailwind的外观设计还是很顺利的。

## 后端部分的收获

这次后端开发是我首次使用 ASP.NET Core 构建 RESTful API。虽然此前接触过 Java Spring，但 C# 的生态和工具链仍带来不少新鲜感。通过实践，我接触到了 Minimal API、依赖注入、数据库迁移、跨域配置以及 API 文档生成等技术部分

### Minimal API

ASP.NET Core 的 Minimal APIs 是一种轻量级 API 开发模式，无需创建控制器类，直接在 Program.cs 中通过方法链定义路由和处理逻辑，极大简化了小型项目的搭建流程

构建RESTfulAPI的过程就是向app的map函数传回调函数

```C#
app.MapGet("foo",async(TaskDb db)=>{
//...处理请求

})
```

在这样的基础下，开发过程是十分简单的

### 配置数据库

配置数据库的过程还是相当简单的，首先我们要配置数据对象

```C#
// TaskItem.cs
public class TaskItem
{
    public int Id { get; set; }
    public required string Title { get; set; }
    public string? Content { get; set; }
    public string? Status { get; set; }
    public string? Owner { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

再去配置我们的针对这种对象的数据库上下文（什么是数据库上下文，见下一节DI容器）

```C#
using Microsoft.EntityFrameworkCore;


// TaskDb继承DbContext
class TaskDb : DbContext
{
    public TaskDb(DbContextOptions<TaskDb> options) : base(options) { }
    public DbSet<TaskItem> Tasks => Set<TaskItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<TaskItem>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Id).ValueGeneratedOnAdd();
        });
    }
}
```

在这里我们指定Id是自增的，防止主键相同发生冲突

### DI容器

就和JavaSpring有一样，C#也有依赖注入，控制反转容器。并且ASP.NET Core 内置了轻量级的依赖注入容器（Microsoft.Extensions.DependencyInjection），无需额外引入 Spring 等框架，开箱即用。

```C#
using Microsoft.Extensions.DependencyInjection;
```

在我看来，IoC容器所做的事就是为各种类自动注入依赖对象并管理对象的生命周期。这样项目的很多上下文依赖都继承特定的上下文部分，然后就可以在容器中注册，由容器来管理依赖和生命周期，简化了管理依赖的困难。

而ASP.NET WebApplication的构建采用构造器模式，先创建构造器，再在构造器中设置各种app的配置，配置完毕后再生成app对象，而依赖管理等这一步就是在构造器中设置的

```C#
var builder = WebApplication.CreateBuilder(args);
// Add services to the container.
// Learn more about configuring OpenAPI at https://aka.ms/aspnet/openapi
// 通过AddDbContext<TaskDb>把TaskDb添加到容器中
builder.Services.AddDbContext<TaskDb>(opt => { opt.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")); });
builder.Services.AddDatabaseDeveloperPageExceptionFilter();
```

而builder.Services就是一个IServiceCollection，我们通过AddDbContext方法将TaskDb设置为我们要使用的数据库上下文依赖，同时设置数据库连接参数为appsettings.json里面的DefaultConnection这个字符串。

### 跨域

当后端写好get方法，从前端访问却总是遇到问题，那就是日常CORS问题，即服务器总是禁止跨域请求

这里我们通过一句代码解决

```C#
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowVueDev", policy =>
    {
        policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod();
        // 生产环境中应限制具体域名，避免安全风险
    });
});
```

### swagger

我使用 Microsoft.AspNetCore.OpenApi 和 Swashbuckle.AspNetCore 自动生成并可视化 API 文档（Swagger UI），并在开发环境启用交互式 UI（类似 Swagger UI），方便调试接口。

### 数据库迁移

通过 EF Core 的迁移功能，我可以将 C# 模型变更同步到数据库。例如，添加新字段后，只需运行

```bash
dotnet ef migrations add AddTaskStatus
dotnet ef database update
```

就可以同步C#模型到数据库，而不用手写sql。

迁移文件是代码化的数据库变更记录，可纳入版本控制，实现数据库版本管理，这种像管理Git项目一样管理数据库结构的做法，初见不可谓不惊艳。

## 思想架构分析

其实架构分析往往不在开发后才分析，而是在开发前就学习，开发后再总结，不过这里省略开发前的学习，直接谈实战接触后的认识了。

### 前端架构

#### 传统MVC模型

先在开始讲现代项目架构之前，先解释一下传统的MVC模型。

MVC是在前后端尚不分离时就有的经典模型，非常经典的例子就是我当初写过的Java Spring+Hibernate+Struts技术栈。Struts 是 MVC 的 Web 层实现，Spring 管理业务逻辑（Service），Hibernate 管理数据访问（DAO），三者共同构成分层架构。

MVC中M指模型，V指视图，而C是控制器

M是后端的具体业务，如数据库的管理等，V是前端显示的视图。MVC的在当初的卓越在于它在前后端不分离(jsp)时代就有了让前端和后端尽可能分开的意识。它要求前端不能直接去访问后端，而是通过控制器Controller去访问后端。前端视图View只能调用Controller，Controller再去调用模型Model中的业务代码处理业务。

#### MVC的缺陷

随着MVC项目的不断发展，就不得不写出大量Controller胶水代码来黏合M和V，同时使得Controller的这对职责过多，容易成为上帝类，对项目管理产生很大的挑战，也降低了开发效率。

#### 前端MVVM模型

现代前端如vue采用的架构叫MVVM架构，它由三个部分M,V,VM构成。
它其实很好理解，但首先让我们抛掉MVC的包袱。

其中M为模型，V为视图，而VM则叫模型视图。

我们先给出综述：

首先MVVM模式强依赖于数据绑定，它的核心思路是M中存着前端业务代码（如各种向后端发送请求的逻辑，前端存有的核心数据等），而VM持有希望暴露给视图的代码，V则是视图呈现。

这里的M，V延续了MVC中的词汇，然而和传统MVC中的MV并不相同。最大的区别就是应用场景从前后端变成了仅前端。这里的M是前端的M，不是后端的M。这句话的意思是，在传统MVC模型中M是后端的核心逻辑(如通过ORM向数据库中添加一名学生)，V是大前端，而前端MVC是单针对于前端说的，这里的 M 是前端的数据模型（如通过 axios 向 /student 发送 POST 请求），而具体的后端持久化逻辑（如 ORM 操作）对前端是透明的。而V仅仅指显示出去的视图。

那么什么是VM呢，View-Model视图模型，其实名字上已经能看出，它就是指视图的模型。我们不直接把模型中的数据给到视图来显示，而是通过把数据给到VM，VM再把数据给到V来显示。

前面说过，MVVM强依赖数据绑定，这正是在VM中体现的，VM会监听 Model 的变化并自动更新 View，同时也处理用户的交互（如点击、输入）并更新 Model。

#### MVVM模型的好处

由于VM通过响应式数据自动更新视图，用户交互通过事件更新VM状态，自动同步M和V的状态，减少了手动操作DOM的样板代码，使代码整洁，降低出错率。

更加模块化，职责更清晰。VM专注业务状态和逻辑，不需直接操作DOM;V只负责展示，通过声明式语法和VM通信；M只包含核心数据和逻辑。

易测试。VM是纯js对象，不依赖DOM，可以方便地做单元测试。

响应式数据天然更适合响应式编程。

### 后端架构

该项目目前后端过于简单，只是参考了RESTful API和Minimal API，实际上还没有采用Clean Architecture架构。还没有划分DTO，所以该博客省略该部分。

未来可引入 DTO（Data Transfer Object） 隔离数据库实体与 API 模型，提升安全性与灵活性。
