---
layout: post
title: "面向类型设计： 发现新概念"
description: "Gaining deeper insight into the domain"
date: 2013-01-15
nav: thinking-functionally
seriesId: "面向类型设计"
seriesOrder: 4
categories: [Types, DDD]
---

在上一篇文章中，我们了解了如何使用类型来表示业务规则。

规则是：*「联系人必须有电子邮件或邮政地址」*。

我们设计的类型是：

```fsharp
type ContactInfo =
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo
```

现在假设企业决定也需要支持电话号码。新的商业规则是：*「联系人必须至少有以下其中之一：电子邮箱、邮政地址、家庭电话或工作电话」*。

我们现在如何表示它呢？

稍加思考就会发现，这四种接触方式有15种可能的组合。我们肯定不想创建一个有15种选择的联合案例吧？还有更好的方法吗？

让我们继续这个想法，看看另一个不同但相关的问题。

## 当需求变更时强制进行重大变更 ##

这就是问题所在。假设你有一个联系人结构，其中包含电子邮件地址列表和邮政地址列表，如下所示：

```fsharp
type ContactInformation =
    {
    EmailAddresses : EmailContactInfo list;
    PostalAddresses : PostalContactInfo list
    }
```

另外，假设你已经创建了一个`printReport`函数，它循环遍历这些信息并将其打印出来：

```fsharp
// 模拟代码
let printEmail emailAddress =
    printfn "Email Address is %s" emailAddress

// 模拟代码
let printPostalAddress postalAddress =
    printfn "Postal Address is %s" postalAddress

let printReport contactInfo =
    let {
        EmailAddresses = emailAddresses;
        PostalAddresses = postalAddresses;
        } = contactInfo
    for email in emailAddresses do
         printEmail email
    for postalAddress in postalAddresses do
         printPostalAddress postalAddress
```

粗糙，但简单易懂。

现在，如果新的业务规则生效，我们可能决定改变结构，为电话号码添加一些新的列表。更新后的结构现在看起来像这样：

```fsharp
type PhoneContactInfo = string // 临时假数据

type ContactInformation =
    {
    EmailAddresses : EmailContactInfo list;
    PostalAddresses : PostalContactInfo list;
    HomePhones : PhoneContactInfo list;
    WorkPhones : PhoneContactInfo list;
    }
```

如果做了这个修改，还需要确保所有处理联系信息的函数也都更新了，以处理新的手机例子。

当然，你将被迫修复任何破坏的模式匹配。但在很多情况下，你*不会*被迫处理新案例。

例如，修改后的`printReport`可用于新列表：

```fsharp
let printReport contactInfo =
    let {
        EmailAddresses = emailAddresses;
        PostalAddresses = postalAddresses;
        } = contactInfo
    for email in emailAddresses do
         printEmail email
    for postalAddress in postalAddresses do
         printPostalAddress postalAddress
```

你能看出这是故意的错误吗？是的，我忘记更改处理电话的功能了。记录中的新字段根本没有导致代码中断。不能保证你会记得处理新案件。这太容易忘记了。

再次，我们面临的挑战是：我们能否设计出这样的类型，使这些情况不会轻易发生？

## 更深入的领域洞察 ##

如果你更深入地思考这个例子，你会意识到我们是只见树木不见森林。

我们最初的概念是：*「要联系客户，将有一个可能的电子邮件列表，以及可能的地址列表，等等」*。

但实际上，这一切都是错误的。一个更好的概念是：*「要联系客户，将有一个联系方法列表。每种联系方式都可以是电子邮件、邮政地址或电话号码」*。

这是对域应该如何建模的关键洞察。它创建了一个全新的类型「ContactMethod」，一下子就解决了我们的问题。

我们可以立即重构类型以使用这个新概念：

```fsharp
type ContactMethod =
    | Email of EmailContactInfo
    | PostalAddress of PostalContactInfo
    | HomePhone of PhoneContactInfo
    | WorkPhone of PhoneContactInfo

type ContactInformation =
    {
    ContactMethods  : ContactMethod list;
    }
```

现在必须修改报告代码以处理新类型：

```fsharp
// 模拟代码
let printContactMethod cm =
    match cm with
    | Email emailAddress ->
        printfn "Email Address is %s" emailAddress
    | PostalAddress postalAddress ->
         printfn "Postal Address is %s" postalAddress
    | HomePhone phoneNumber ->
        printfn "Home Phone is %s" phoneNumber
    | WorkPhone phoneNumber ->
        printfn "Work Phone is %s" phoneNumber

let printReport contactInfo =
    let {
        ContactMethods=methods;
        } = contactInfo
    methods
    |> List.iter printContactMethod
```

这些变化有很多好处。

首先，从建模的角度来看，新类型更好地表示了领域，并且更能适应变化的需求。

从开发的角度来看，将类型更改为联合意味着我们添加（或删除）的任何新情况都将以非常明显的方式破坏代码，并且将更加难以意外忘记处理所有情况。

{{< book_page_ddd >}}

## 回到具有15种可能组合的业务规则 ##

现在回到最初的例子。我们认为，为了编码业务规则，我们可能必须创建15种不同联系方法的可能组合。

但是报告问题的新见解也影响了我们对业务规则的理解。

有了头脑中的「联系方法」概念，我们可以将需求重新划分为：*「客户必须至少有一种联系方法。联系方式可以是电子邮件、邮政地址或电话号码」*。

下面重新设计`Contact`类型，让它包含一个联系人方法列表：

```fsharp
type Contact =
    {
    Name: PersonalName;
    ContactMethods: ContactMethod list;
    }
```

但这仍然不完全正确。列表可以是空的。我们如何执行必须*至少*有一个联系方法的规则？

最简单的方法是创建一个必填字段，如下所示：

```fsharp
type Contact =
    {
    Name: PersonalName;
    PrimaryContactMethod: ContactMethod;
    SecondaryContactMethods: ContactMethod list;
    }
```

在这个设计中，`PrimaryContactMethod`是必需的，而次要联系方法是可选的，这正是业务规则所要求的！

这种重构也给了我们一些启示。「主要」和「次要」联系方法的概念可能反过来澄清其他领域的代码，创建洞察力和重构的级联变化。

## 总结 ##

在这篇文章中，我们看到了如何使用类型来建模业务规则，实际上可以帮助你在更深层次上理解领域。

在《领域驱动设计》（*Domain Driven Design*）一书中，Eric Evans用了整整一节，特别是两章（第8章和第9章）来讨论[重构对于深入洞察](http://dddcommunity.org/wp-content/uploads/files/books/evans_pt03.pdf)的重要性。相比之下，这篇文章中的例子很简单，但我希望它表明这样的洞察力可以帮助提高模型和代码的正确性。

在下一篇文章中，我们将看到类型如何帮助表示细粒度状态。
