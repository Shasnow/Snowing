---
title: POCO（Plain Old CLR Object）简介
published: 2026-03-22
pinned: false
description: 介绍 POCO（Plain Old CLR Object）的概念、特征以及在现代 C# 开发中的应用，特别是在领域驱动设计（DDD）和 ORM（如 Entity Framework Core）中的重要性。
tags: [ POCO, C#, ORM ]
category: C#
licenseName: "Unlicensed"
draft: false
---

POCO 是 Plain Old CLR Object 的缩写，直译为“简单的旧 CLR 对象”。

这个术语借鉴于 Java 世界的 POJO (Plain Old Java Object)。它的核心思想是：一个纯粹的、简单的 .NET 对象，不依赖于任何特定的框架、基类或接口。

1. POCO 的核心特征
   一个标准的 POCO 类通常具有以下特征：
    1. 没有继承特定基类：它不继承自像 MarshalByRefObject、Entity、Controller 这样的框架特定基类。通常直接继承自
       System.Object（虽然在 C# 中这通常是隐式的）。
    2. 没有实现特定接口：它不需要实现像 INotifyPropertyChanged、IDataErrorInfo 或 IEntity 这样的接口（尽管在实际项目中为了方便，有时会妥协实现这些接口）。
    3. 仅包含数据：主要由属性（Properties）、字段（Fields）和简单的业务逻辑（如验证）组成。
    4. 持久化无关（Persistence Ignorant）：POCO 类不知道数据是如何存储的（是存到 SQL Server、MySQL 还是 XML 文件）。它只是数据的容器。

2. 代码示例

   ✅ 这是一个标准的 POCO 类：

    ```csharp
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
        public DateTime CreatedDate { get; set; }
    
        // 可以包含简单的业务逻辑，但不涉及数据库或UI
        public bool IsValidEmail()
        {
            return Email.Contains("@");
        }
    }
    ```

   **特点**：

    - 只有自动实现的属性。
        - 没有继承任何东西。
        - 没有实现任何接口。
        - 可以在任何地方（控制台、Web、桌面应用）使用，不依赖 ASP.NET 或 Entity Framework。

   ❌ 这不是 POCO 类（依赖框架）：
    ```csharp
    // 依赖 Entity Framework Core
    public class User : EntityBase 
    {
        public int Id { get; set; }
        public string Name { get; set; }
    }
    
    // 依赖 WPF/WinForms 数据绑定
    public class User : INotifyPropertyChanged 
    {
        private string _name;
        public string Name 
        {
            get => _name;
            set { _name = value; OnPropertyChanged(); }
        }
        // ... 大量的 INotifyPropertyChanged 实现代码
    }
    ```

3. 为什么要使用 POCO？
    1. 解耦（Decoupling）：
        - 你的业务逻辑类不依赖于数据库框架（如 EF Core）或 UI 框架（如 WPF/Blazor）。
        - 如果你想从 Entity Framework 换成 Dapper，或者从 MVC 换成 Minimal API，POCO 类完全不需要修改。
    2. 可测试性（Testability）：
        - 因为 POCO 不依赖外部框架，你可以非常容易地在单元测试中实例化它们并进行断言，而不需要模拟复杂的框架上下文。
    3. 简单性：
        - 代码更干净，更容易理解。它就是数据的“形状”，没有任何魔法。
    4. ORM 友好：
        - 现代 ORM（如 Entity Framework Core）都推荐使用 POCO 作为实体类（Entity）。EF Core 通过约定（Convention）或 Fluent API
          来映射
          POCO 到数据库表，而不需要 POCO 本身做任何特殊处理。
4. POCO vs DTO (Data Transfer Object)
   这两个概念经常一起出现，但有细微区别：
    - POCO (领域模型/实体)：
        - 通常对应数据库中的表。
        - 可能包含业务逻辑（如计算、验证）。
        - 例如：User 实体，可能有 ChangePassword() 方法。
    - DTO (数据传输对象)：
        - 专门用于在不同层（如 Controller 和 View，或 微服务之间）传输数据。
        - 绝对没有业务逻辑，只有属性。
        - 形状往往是为特定视图裁剪的（例如 UserCreateDto 可能没有 Id 字段）。
          注意：在很多简单的项目中，POCO 和 DTO 是同一个类，但在复杂的领域驱动设计（DDD）中，它们是严格分开的。

5. 现代 C# 中的 POCO：Record
   在 C# 9 及更高版本中，引入了 record 类型，它是创建 POCO 的完美语法糖：

```csharp
// 这就是一个不可变的 POCO
public record User(int Id, string Name, string Email);
```

总结
POCO 就是一个“干净”的 C# 类，它只关心“我有什么数据”，而不关心“数据怎么存”或“数据怎么显示”。 它是 .NET 中实现松耦合架构的基石。