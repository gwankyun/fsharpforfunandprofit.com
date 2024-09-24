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

But really, this is all wrong. A much better concept is: *"To contact a customer, there will be a list of contact methods. Each contact method could be an email OR a postal address OR a phone number"*.

This is a key insight into how the domain should be modelled.  It creates a whole new type, a "ContactMethod", which resolves our problems in one stroke.

We can immediately refactor the types to use this new concept:

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

And the reporting code must now be changed to handle the new type as well:

```fsharp
// mock code
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

These changes have a number of benefits.

First, from a modelling point of view, the new types represent the domain much better, and are more adaptable to changing requirements.

And from a development point of view, changing the type to be a union means that any new cases that we add (or remove) will break the code in a very obvious way, and it will be much harder to accidentally forget to handle all the cases.

{{< book_page_ddd >}}

## Back to the business rule with 15 possible combinations ##

So now back to the original example. We left it thinking that, in order to encode the business rule, we might have to create 15 possible combinations of various contact methods.

But the new insight from the reporting problem also affects our understanding of the business rule.

With the "contact method" concept in our heads, we can rephase the requirement as: *"A customer must have at least one contact method. A contact method could be an email OR a postal addresses OR a phone number"*.

So let's redesign the `Contact` type to have a list of contact methods:

```fsharp
type Contact =
    {
    Name: PersonalName;
    ContactMethods: ContactMethod list;
    }
```

But this is still not quite right. The list could be empty.  How can we enforce the rule that there must be *at least* one contact method?

The simplest way is to create a new field that is required, like this:

```fsharp
type Contact =
    {
    Name: PersonalName;
    PrimaryContactMethod: ContactMethod;
    SecondaryContactMethods: ContactMethod list;
    }
```

In this design, the `PrimaryContactMethod` is required, and the secondary contact methods are optional, which is exactly what the business rule requires!

And this refactoring too, has given us some insight.  It may be that the concepts of "primary" and "secondary" contact methods might, in turn, clarify code in other areas, creating a cascading change of insight and refactoring.

## Summary ##

In this post, we've seen how using types to model business rules can actually help you to understand the domain at a deeper level.

In the *Domain Driven Design* book, Eric Evans devotes a whole section and two chapters in particular (chapters 8 and 9) to discussing the importance of [refactoring towards deeper insight](http://dddcommunity.org/wp-content/uploads/files/books/evans_pt03.pdf).  The example in this post is simple in comparison, but I hope that it shows that how an insight like this can help improve both the model and the code correctness.

In the next post, we'll see how types can help with representing fine-grained states.
