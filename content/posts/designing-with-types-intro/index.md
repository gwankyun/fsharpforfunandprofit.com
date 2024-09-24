---
layout: post
title: "面向类型设计： 引言"
description: "Making design more transparent and improving correctness"
date: 2013-01-12
nav: thinking-functionally
seriesId: "面向类型设计"
seriesOrder: 1
categories: [Types, DDD]
---

在本系列中，我们将介绍在设计过程中使用类型的一些方法。特别是，深思熟虑地使用类型可以使设计更加透明，同时提高正确性。

本系列将关注设计的「微观层面」。也就是说，在单个类型和函数的最低级别上工作。更高层次的设计方法，以及使用函数式或面向对象风格的相关决策，将在另一个系列中讨论。

许多建议在C\#或Java中也是可行的，但是F\#类型的轻量级特性意味着我们更有可能进行这种重构。

## 一个基本例子 ##

为了演示类型的各种用法，我将使用一个非常简单的示例，即`Contact`类型，如下所示。

```fsharp
type Contact =
    {
    FirstName: string;
    MiddleInitial: string;
    LastName: string;

    EmailAddress: string;
    // 如果确认了电子邮件地址的所有权，返回true
    IsEmailVerified: bool;

    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    // 如果已根据地址服务进行验证，则为true
    IsAddressValid: bool;
    }

```

这似乎很明显——我相信我们都见过很多次这样的事情。那么我们能做些什么呢？我们如何重构它以充分利用类型系统呢？

## 创建「原子」类型 ##

首先要做的是查看数据访问和更新的使用模式。例如，是否有可能在更新`Zip`的同时不更新`Address1` ？另一方面，事务更新`EmailAddress`而不更新`FirstName`可能很常见。

这就引出了第一条准则：

* *指导原则：使用记录或元组将需要保持一致性的数据（即「原子」）分组在一起，但不要不必要地将不相关的数据分组在一起。*

在本例中，很明显三个名称值是一个集合，地址值是一个集合，电子邮件也是一个集合。

我们在这里也有一些额外的标志，如`IsAddressValid`和`IsEmailVerified`。这些应该是相关集合的一部分吗？现在当然可以，因为标志依赖于相关的值。

例如，如果`EmailAddress`更改，那么`IsEmailVerified`可能需要同时重置为false。

对于`PostalAddress`，很明显，核心「address」部分是一个有用的公共类型，没有`IsAddressValid`标志。另一方面，`IsAddressValid`与地址相关联，当地址发生变化时，`IsAddressValid`将被更新。

因此，我们似乎应该创建*两个*类型。一个是通用的`PostalAddress`，另一个是联系人上下文中的地址，我们可以称其为`PostalContactInfo`。

```fsharp
type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    }

type PostalContactInfo =
    {
    Address: PostalAddress;
    IsAddressValid: bool;
    }
```

最后，我们可以使用选项类型来表示某些值，如`MiddleInitial`，确实是可选的。

```fsharp
type PersonalName =
    {
    FirstName: string;
    // 使用「option」来表示可选
    MiddleInitial: string option;
    LastName: string;
    }
```

## 总结 ##

经过这些修改，我们现在有了以下代码：

```fsharp
type PersonalName =
    {
    FirstName: string;
    // 使用「option」来表示可选
    MiddleInitial: string option;
    LastName: string;
    }

type EmailContactInfo =
    {
    EmailAddress: string;
    IsEmailVerified: bool;
    }

type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    }

type PostalContactInfo =
    {
    Address: PostalAddress;
    IsAddressValid: bool;
    }

type Contact =
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }

```

我们还没有编写一个函数，但是代码已经更好地表示了域。然而，这只是我们所能做的事情的开始。

接下来，使用单例联合为基本类型添加语义。
