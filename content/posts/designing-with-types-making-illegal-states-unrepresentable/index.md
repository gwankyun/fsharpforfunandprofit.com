---
layout: post
title: "Designing with types: Making illegal states unrepresentable"
description: "Encoding business logic in types"
date: 2013-01-14
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 3
categories: [Types, DDD]
---

在这篇文章中，我们来看看F\#的一个关键好处，它使用类型系统来「使非法状态不可表示」（一个借用自Yaron Minsky的短语）。

来看看我们的`Contact`类型。多亏了前面的重构，它变得非常简单：

```fsharp
type Contact =
    {
    Name: Name;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }
```

现在假设我们有以下简单的业务规则：*「联系人必须有电子邮件或邮政地址」*。我们的类型符合这个规则吗？

答案是否定的。业务规则意味着联系人可能有电子邮件地址但没有邮政地址，反之亦然。但就目前而言，我们的类型要求联系人必须始终*都具有*这两项信息。

答案似乎很明显——将地址设为可选，如下所示：

```fsharp
type Contact =
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo option;
    PostalContactInfo: PostalContactInfo option;
    }
```

但现在我们在另一条路上走得太远了。在这种设计中，一个联系人可能没有任何类型的地址。但是业务规则要求*必须*提供至少一条信息。

解决方案是什么？

## 使非法状态不可表示 ##

如果我们仔细考虑业务规则，我们意识到有三种可能性：

* 联系人只有电子邮件地址
* 联系人只有一个邮政地址
* 联系人有电子邮件地址和邮政地址

一旦它变成这样，解决方案就变得显而易见了——为每种可能性使用联合类型。

```fsharp
type ContactInfo =
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo

type Contact =
    {
    Name: Name;
    ContactInfo: ContactInfo;
    }
```

这个设计完全符合要求。这三种情况都明确表示，第四种可能的情况（根本没有电子邮件或邮政地址）是不允许的。

注意，对于「电子邮件地址和邮政地址」的情况，我现在只使用元组类型。完全可以满足我们的需要。

### 构造ContactInfo ###

现在让我们看看如何在实践中使用它。首先创建一个新的联系人：

```fsharp
let contactFromEmail name emailStr =
    let emailOpt = EmailAddress.create emailStr
    // 处理邮件有效或无效的情况
    match emailOpt with
    | Some email ->
        let emailContactInfo =
            {EmailAddress=email; IsEmailVerified=false}
        let contactInfo = EmailOnly emailContactInfo
        Some {Name=name; ContactInfo=contactInfo}
    | None -> None

let name = {FirstName = "A"; MiddleInitial=None; LastName="Smith"}
let contactOpt = contactFromEmail name "abc@example.com"
```

在这段代码中，我们创建了一个简单的辅助函数`contactFromEmail`，通过传入姓名和电子邮件地址来创建一个新联系人。然而，电子邮件可能是无效的，因此这个函数必须处理这两种情况，它返回一个`Contact option`，而不是`Contact`。

### 更新ContactInfo ###

现在，如果需要在已有的`ContactInfo`中添加邮政地址，我们别无选择，只能同时处理这三种情况：

* 如果联系人以前只有电子邮件地址，现在它同时有电子邮件地址和邮政地址，因此使用`EmailAndPost`实例返回一个联系人。
* 如果联系人以前只有邮政地址，则使用`PostOnly`返回联系人，替换现有地址。
* 如果联系人之前同时拥有电子邮件地址和邮政地址，则使用`EmailAndPost`返回一个联系人，替换现有的地址。

这是一个更新邮政地址的辅助方法。你可以看到它是如何显式地处理每种情况的。

```fsharp
let updatePostalAddress contact newPostalAddress =
    let {Name=name; ContactInfo=contactInfo} = contact
    let newContactInfo =
        match contactInfo with
        | EmailOnly email ->
            EmailAndPost (email,newPostalAddress)
        | PostOnly _ -> // 忽略现有地址
            PostOnly newPostalAddress
        | EmailAndPost (email,_) -> // 忽略现有地址
            EmailAndPost (email,newPostalAddress)
    // 新建一个联系人
    {Name=name; ContactInfo=newContactInfo}
```

下面是使用的代码：

```fsharp
let contact = contactOpt.Value   // 请参阅关于选项的警告。下面的值
let newPostalAddress =
    let state = StateCode.create "CA"
    let zip = ZipCode.create "97210"
    {
        Address =
            {
            Address1= "123 Main";
            Address2="";
            City="Beverly Hills";
            State=state.Value; // 请参阅关于选项的警告。下面的值
            Zip=zip.Value;     // 请参阅关于选项的警告。下面的值
            };
        IsAddressValid=false
    }
let newContact = updatePostalAddress contact newPostalAddress
```

*警告:在这段代码中，我使用`option.Value`来提取选项的内容。这在交互式环境下是可以的，但在生产代码中是非常糟糕的做法！你应该始终使用匹配来处理选项的两种情况。*

## 为什么要费心创建这些复杂的类型呢？ ##

说到这里，你可能会说我们让事情变得不必要的复杂了。我将以以下几点来回答：

首先，业务逻辑*很*复杂。没有简单的方法可以避免它。如果你的代码没有这么复杂，说明你没有正确地处理所有情况。

其次，如果逻辑是由类型表示的，那么它是自动的自文档。你可以查看下面的联合示例，并立即了解业务规则是什么。你不必花费任何时间试图分析任何其他代码。

```fsharp
type ContactInfo =
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo
```

最后，如果逻辑由类型表示，对业务规则的任何更改都将立即创建重大更改，这通常是一件好事。

在下一篇文章中，我们将深入探讨最后一点。当你尝试使用类型表示业务逻辑时，你可能会突然发现，可以获得对该领域的全新见解。

{{< book_page_ddd_img >}}
