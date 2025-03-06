---
layout: post
title: "面向类型设计： 明确状态"
description: "Using state machines to ensure correctness"
date: 2013-01-16
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 5
categories: [Types, DDD]
---

在这篇文章中，我们将探讨如何通过使用状态机来明确隐式状态，然后使用联合类型对这些状态机进行建模。

## 背景 ##

在本系列的[上一篇文章](/posts/designing-with-types-single-case-dus/)中，我们研究了单例联合类型，将其作为诸如电子邮件地址等类型的包装器。

```fsharp
module EmailAddress =

    type T = EmailAddress of string

    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then Some (EmailAddress s)
            else None
```

这段代码假设一个地址要么有效，要么无效。如果无效，我们会完全拒绝它，并返回`None`而不是一个有效的值。

但有效性是有程度之分的。例如，如果我们想保留一个无效的电子邮件地址而不是直接拒绝它，该怎么办呢？在这种情况下，和往常一样，我们希望使用类型系统来确保不会将有效地址与无效地址混淆。

显而易见的做法是使用联合类型：

```fsharp
module EmailAddress =

    type T =
        | ValidEmailAddress of string
        | InvalidEmailAddress of string

    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then ValidEmailAddress s    // 更改结果类型
            else InvalidEmailAddress s  // 更改结果类型

    // test
    let valid = create "abc@example.com"
    let invalid = create "example.com"
```

有了这些类型，我们可以确保只发送有效的电子邮件：

```fsharp
let sendMessageTo t =
    match t with
    | ValidEmailAddress email ->
         // 发送电子邮件
    | InvalidEmailAddress _ ->
         // 忽略
```

到目前为止，一切都很顺利。到现在为止，这种设计对你来说应该是显而易见的。

但这种方法的应用范围比你想象的要广泛。在许多情况下，存在类似的“状态”，但这些状态没有被明确表示出来，而是在代码中用标志、枚举或条件逻辑来处理。

## 状态机 ##

在上面的例子中，“有效”和“无效”的情况是相互排斥的。也就是说，一个有效的电子邮件永远不会变成无效的，反之亦然。

但在很多情况下，通过某种事件的触发，从一种情况转变到另一种情况是可能的。这时我们就有了一个[“状态机”](http://en.wikipedia.org/wiki/Finite-state_machine)，其中每种情况代表一个“状态”，从一个状态到另一个状态的转变就是一个“转换”。

一些例子：

* 一个电子邮件地址可能有“未验证”和“已验证”两种状态，你可以通过要求用户点击确认邮件中的链接，从“未验证”状态转换到“已验证”状态。
![State transition diagram: Verified Email](./State_VerifiedEmail.png)

* 一个购物车可能有“空”、“活跃”和“已付款”三种状态，你可以通过向购物车中添加商品，从“空”状态转换到“活跃”状态，通过付款转换到“已付款”状态。
![State transition diagram: Shopping Cart](./State_ShoppingCart.png)

* 像国际象棋这样的游戏可能有“白方回合”、“黑方回合”和“游戏结束”三种状态，你可以通过白方进行非结束游戏的移动，从“白方回合”状态转换到“黑方回合”状态，或者通过将死对方的移动转换到“游戏结束”状态。
![State transition diagram: Chess game](./State_Chess.png)

在这些例子中，我们都有一组状态、一组转换，以及可以触发转换的事件。
状态机通常用一个表格来表示，比如下面这个购物车的表格：

{{<rawtable>}}
<table class="table table-condensed">
<thead>
<tr>
<th>Current State</th>
<th>Event-></th>
<th>Add Item</th>
<th>Remove Item</th>
<th>Pay</th>
</tr>
</thead>
<tbody>
<tr>
<th>Empty</th>
<td></td>
<td>new state = Active</td>
<td>n/a</td>
<td>n/a</td>
</tr>
<tr>
<th>Active</th>
<td></td>
<td>new state = Active</td>
<td>new state = Active or Empty,<br> depending on the number of items</td>
<td>new state = Paid</td>
</tr>
<tr>
<th>Paid</th>
<td></td>
<td>n/a</td>
<td>n/a</td>
<td>n/a</td>
</tr>
</tbody>
</table>
{{</rawtable>}}

有了这样的表格，你可以快速了解当系统处于给定状态时，每个事件应该发生什么。

{{< linktarget "why-use" >}}

## 为什么使用状态机？ ##

在这些情况下使用状态机有很多好处：

**每个状态可以有不同的允许行为。**

在已验证电子邮件的例子中，可能有一个业务规则规定，你只能向已验证的电子邮件地址发送密码重置邮件，而不能向未验证的地址发送。
在购物车的例子中，只有活跃的购物车才能付款，已付款的购物车不能再添加商品。

**所有状态都有明确的文档记录。**

很容易出现一些重要的状态是隐式的，但从未被记录下来的情况。

例如，“空购物车”的行为与“活跃购物车”不同，但在代码中很少会明确记录这一点。

**它是一种设计工具，迫使你考虑每一种可能性。**

错误的一个常见原因是某些边缘情况没有得到处理，但状态机迫使你考虑所有情况。

例如，如果我们尝试验证一个已经验证过的电子邮件会发生什么？
如果我们尝试从一个空购物车中移除商品会发生什么？
如果在“黑方回合”时白方尝试移动会发生什么？等等。

## 如何在F#中实现简单的状态机 ##

你可能熟悉复杂的状态机，比如用于语言解析器和正则表达式的状态机。那些类型的状态机是从规则集或语法生成的，非常复杂。

我所说的状态机要简单得多。最多只有几种情况，转换的数量也很少，所以我们不需要使用复杂的生成器。

那么，实现这些简单状态机的最佳方法是什么呢？

通常，每个状态都会有自己的类型，用于存储与该状态相关的数据（如果有的话），然后整个状态集将由一个联合类型表示。

下面是一个使用购物车状态机的例子：

```fsharp
type ActiveCartData = { UnpaidItems: string list }
type PaidCartData = { PaidItems: string list; Payment: float }

type ShoppingCart =
    | EmptyCart  // 无数据
    | ActiveCart of ActiveCartData
    | PaidCart of PaidCartData
```

请注意，`EmptyCart`状态没有数据，因此不需要特殊类型。

每个事件都由一个函数表示，该函数接受整个状态机（联合类型）并返回一个新的状态机版本（同样是联合类型）。

以下是使用购物车的两个事件的示例：

```fsharp
let addItem cart item =
    match cart with
    | EmptyCart ->
        // 创建一个包含一个商品的新活跃购物车
        ActiveCart {UnpaidItems=[item]}
    | ActiveCart {UnpaidItems=existingItems} ->
        // 创建一个添加了商品的新活跃购物车
        ActiveCart {UnpaidItems = item :: existingItems}
    | PaidCart _ ->
        // 忽略
        cart

let makePayment cart payment =
    match cart with
    | EmptyCart ->
        // 忽略
        cart
    | ActiveCart {UnpaidItems=existingItems} ->
        // 创建一个包含付款信息的新已付款购物车
        PaidCart {PaidItems = existingItems; Payment=payment}
    | PaidCart _ ->
        // 忽略
        cart
```

从调用者的角度来看，状态集被视为“一个整体”进行通用操作（`ShoppingCart`类型），但在内部处理事件时，每个状态都被单独处理。

### 设计事件处理函数 ###

准则：*事件处理函数应该始终接受并返回整个状态机*

你可能会问：为什么我们必须将整个购物车传递给事件处理函数？例如，`makePayment`事件仅在购物车处于活跃状态时才有意义，那么为什么不直接传递ActiveCart类型，如下所示：

```fsharp
let makePayment2 activeCart payment =
    let {UnpaidItems=existingItems} = activeCart
    {PaidItems = existingItems; Payment=payment}
```

Let's compare the function signatures:

```fsharp
// 原始函数
val makePayment : ShoppingCart -> float -> ShoppingCart

// 新的更具体函数
val makePayment2 :  ActiveCartData -> float -> PaidCartData
```

你会发现，原始的`makePayment`函数接受一个购物车并返回一个购物车，而新函数接受一个`ActiveCartData`并返回一个 `PaidCartData`，这似乎更相关。

但如果这样做，当购物车处于不同状态（如空或已付款）时，如何处理相同的事件呢？必须在某个地方处理所有三种可能状态的事件，将这种业务逻辑封装在函数内部比依赖调用者要好得多

### 处理“原始”状态 ###

偶尔，你确实需要将其中一个状态视为独立的实体并独立使用它。因为每个状态也是一种类型，所以通常这很简单。

例如，如果我需要报告所有已付款的购物车，我可以传递一个`PaidCartData`列表。

```fsharp
let paymentReport paidCarts =
    let printOneLine {Payment=payment} =
        printfn "Paid %f for items" payment
    paidCarts |> List.iter printOneLine
```

通过使用`PaidCartData`列表作为参数而不是`ShoppingCart`本身，我确保不会意外报告未付款的购物车。

如果这样做，应该在事件处理程序的辅助函数中进行，而不是在事件处理程序本身中。

{{< linktarget "replace-flags" >}}

## Using explicit states to replace boolean flags ##

Let's look at how we can apply this approach to a real example now.

In the `Contact` example from an [earlier post](/posts/designing-with-types-intro/) we had a flag that was used to indicate whether a customer had verified their email address.
The type looked like this:

```fsharp
type EmailContactInfo =
    {
    EmailAddress: EmailAddress.T;
    IsEmailVerified: bool;
    }
```

Any time you see a flag like this, chances are you are dealing with state. In this case, the boolean is used to indicate that we have two states: "Unverified" and "Verified".

As mentioned above, there will probably be various business rules associated with what is permissible in each state. For example, here are two:

* Business rule: *"Verification emails should only be sent to customers who have unverified email addresses"*
* Business rule: *"Password reset emails should only be sent to customers who have verified email addresses"*

As before, we can use types to ensure that code conforms to these rules.

Let's rewrite the `EmailContactInfo` type using a state machine. We'll put it in an module as well.

We'll start by defining the two states.

* For the "Unverified" state, the only data we need to keep is the email address.
* For the "Verified" state, we might want to keep some extra data in addition to the email address, such as the date it was verified, the number of recent password resets, on so on. This data is not relevant (and should not even be visible) to the "Unverified" state.

```fsharp
module EmailContactInfo =
    open System

    // placeholder
    type EmailAddress = string

    // UnverifiedData = just the email
    type UnverifiedData = EmailAddress

    // VerifiedData = email plus the time it was verified
    type VerifiedData = EmailAddress * DateTime

    // set of states
    type T =
        | UnverifiedState of UnverifiedData
        | VerifiedState of VerifiedData

```

Note that for the `UnverifiedData` type I just used a type alias. No need for anything more complicated right now, but using a type alias makes the purpose explicit and helps with refactoring.

Now let's handle the construction of a new state machine, and then the events.

* Construction *always* results in an unverified email, so that is easy.
* There is only one event that transitions from one state to another: the "verified" event.

```fsharp
module EmailContactInfo =

    // types as above

    let create email =
        // unverified on creation
        UnverifiedState email

    // handle the "verified" event
    let verified emailContactInfo dateVerified =
        match emailContactInfo with
        | UnverifiedState email ->
            // construct a new info in the verified state
            VerifiedState (email, dateVerified)
        | VerifiedState _ ->
            // ignore
            emailContactInfo
```

Note that, as [discussed here](/posts/match-expression/), every branch of the match must return the same type, so when ignoring the verified state we must still return something, such as the object that was passed in.

Finally, we can write the two utility functions `sendVerificationEmail` and `sendPasswordReset`.

```fsharp
module EmailContactInfo =

    // types and functions as above

    let sendVerificationEmail emailContactInfo =
        match emailContactInfo with
        | UnverifiedState email ->
            // send email
            printfn "sending email"
        | VerifiedState _ ->
            // do nothing
            ()

    let sendPasswordReset emailContactInfo =
        match emailContactInfo with
        | UnverifiedState email ->
            // ignore
            ()
        | VerifiedState _ ->
            // ignore
            printfn "sending password reset"
```

{{< book_page_ddd >}}


## Using explicit cases to replace case/switch statements ##

Sometimes it is not just a simple boolean flag that is used to indicate state.  In C# and Java it is common to use a `int` or an `enum` to represent a set of states.

For example, here's a simple state diagram of a package status for a delivery system, where a package has three possible states:

![State transition diagram: Package Delivery](./State_Delivery.png)

There are some obvious business rules that come out of this diagram:

* *Rule: "You can't put a package on a truck if it is already out for delivery"*
* *Rule: "You can't sign for a package that is already delivered"*

and so on.

Now, without using union types, we might represent this design by using an enum to represent the state, like this:

```fsharp
open System

type PackageStatus =
    | Undelivered
    | OutForDelivery
    | Delivered

type Package =
    {
    PackageId: int;
    PackageStatus: PackageStatus;
    DeliveryDate: DateTime;
    DeliverySignature: string;
    }
```

And then the code to handle the "putOnTruck" and "signedFor" events might look like this:

```fsharp
let putOnTruck package =
    {package with PackageStatus=OutForDelivery}

let signedFor package signature =
    let {PackageStatus=packageStatus} = package
    if (packageStatus = Undelivered)
    then
        failwith "package not out for delivery"
    else if (packageStatus = OutForDelivery)
    then
        {package with
            PackageStatus=OutForDelivery;
            DeliveryDate = DateTime.UtcNow;
            DeliverySignature=signature;
            }
    else
        failwith "package already delivered"
```

This code has some subtle bugs in it.

* When handling the "putOnTruck" event, what should happen in the case that the status is *already* `OutForDelivery` or `Delivered`. The code is not explicit about it.
* When handling the "signedFor" event, we do handle the other states, but the last else branch assumes that we only have three states, and therefore doesn't bother to be explicit about testing for it. This code would be incorrect if we ever added a new status.
* Finally, because the `DeliveryDate` and `DeliverySignature` are in the basic structure, it would be possible to set them accidentally, even though the status was not `Delivered`.

But as usual, the idiomatic and more type-safe F# approach is to use an overall union type rather than embed a status value inside a data structure.

```fsharp
open System

type UndeliveredData =
    {
    PackageId: int;
    }

type OutForDeliveryData =
    {
    PackageId: int;
    }

type DeliveredData =
    {
    PackageId: int;
    DeliveryDate: DateTime;
    DeliverySignature: string;
    }

type Package =
    | Undelivered of UndeliveredData
    | OutForDelivery of OutForDeliveryData
    | Delivered of DeliveredData
```

And then the event handlers *must* handle every case.

```fsharp
let putOnTruck package =
    match package with
    | Undelivered {PackageId=id} ->
        OutForDelivery {PackageId=id}
    | OutForDelivery _ ->
        failwith "package already out"
    | Delivered _ ->
        failwith "package already delivered"

let signedFor package signature =
    match package with
    | Undelivered _ ->
        failwith "package not out"
    | OutForDelivery {PackageId=id} ->
        Delivered {
            PackageId=id;
            DeliveryDate = DateTime.UtcNow;
            DeliverySignature=signature;
            }
    | Delivered _ ->
        failwith "package already delivered"
```

*Note: I am using `failWith` to handle the errors. In a production system, this code should be replaced by client driven error handlers.
See the discussion of handling constructor errors in the [post about single case DUs](/posts/designing-with-types-single-case-dus/) for some ideas.*

## Using explicit cases to replace implicit conditional code ##

Finally, there are often cases where a system has states, but they are implicit in conditional code.

For example, here is a type that represents an order.

```fsharp
open System

type Order =
    {
    OrderId: int;
    PlacedDate: DateTime;
    PaidDate: DateTime option;
    PaidAmount: float option;
    ShippedDate: DateTime option;
    ShippingMethod: string option;
    ReturnedDate: DateTime option;
    ReturnedReason: string option;
    }
```

You can guess that Orders can be "new", "paid", "shipped" or "returned", and have timestamps and extra information for each transition, but this is not made explicit in the structure.

The option types are a clue that this type is trying to do too much.  At least F# forces you to use options -- in C# or Java these might just be nulls, and you would have no idea from the type definition whether they were required or not.

And now let's look at the kind of ugly code that might test these option types to see what state the order is in.

Again, there is some important business logic that depends on the state of the order, but nowhere is it explicitly documented what the various states and transitions are.

```fsharp
let makePayment order payment =
    if (order.PaidDate.IsSome)
    then failwith "order is already paid"
    //return an updated order with payment info
    {order with
        PaidDate=Some DateTime.UtcNow
        PaidAmount=Some payment
        }

let shipOrder order shippingMethod =
    if (order.ShippedDate.IsSome)
    then failwith "order is already shipped"
    //return an updated order with shipping info
    {order with
        ShippedDate=Some DateTime.UtcNow
        ShippingMethod=Some shippingMethod
        }
```

*Note: I added `IsSome` to test for option values being present as a direct port of the way that a C# program would test for `null`. But `IsSome` is both ugly and dangerous.  Don't use it!*

Here is a better approach using types that makes the states explicit.

```fsharp
open System

type InitialOrderData =
    {
    OrderId: int;
    PlacedDate: DateTime;
    }
type PaidOrderData =
    {
    Date: DateTime;
    Amount: float;
    }
type ShippedOrderData =
    {
    Date: DateTime;
    Method: string;
    }
type ReturnedOrderData =
    {
    Date: DateTime;
    Reason: string;
    }

type Order =
    | Unpaid of InitialOrderData
    | Paid of InitialOrderData * PaidOrderData
    | Shipped of InitialOrderData * PaidOrderData * ShippedOrderData
    | Returned of InitialOrderData * PaidOrderData * ShippedOrderData * ReturnedOrderData
```

And here are the event handling methods:

```fsharp
let makePayment order payment =
    match order with
    | Unpaid i ->
        let p = {Date=DateTime.UtcNow; Amount=payment}
        // return the Paid order
        Paid (i,p)
    | _ ->
        printfn "order is already paid"
        order

let shipOrder order shippingMethod =
    match order with
    | Paid (i,p) ->
        let s = {Date=DateTime.UtcNow; Method=shippingMethod}
        // return the Shipped order
        Shipped (i,p,s)
    | Unpaid _ ->
        printfn "order is not paid for"
        order
    | _ ->
        printfn "order is already shipped"
        order
```

*Note: Here I am using `printfn` to handle the errors. In a production system, do use a different approach.*


## When not to use this approach

As with any technique we learn, we have to be careful of treating it like a [golden hammer](http://en.wikipedia.org/wiki/Law_of_the_instrument).

This approach does add complexity, so before you start using it, be sure that benefits will outweigh the costs.

To recap, here are the conditions where using simple state machines might be benficial:

* You have a set of mutually exclusive states with transitions between them.
* The transitions are triggered by external events.
* The states are exhaustive. That is, there are no other choices and you must always handle all cases.
* Each state might have associated data that should not be accessible when the system is in another state.
* There are static business rules that apply to the states.

Let's look at some examples where these guidelines *don't* apply.

**States are not important in the domain.**

Consider a blog authoring application. Typically, each blog post can be in a state such as "Draft", "Published", etc. And there are obviously transitions between these states driven by events (such as clicking a "publish" button).

But is it worth creating a state machine for this? Generally, I would say not.

Yes, there are state transitions, but is there really any change in logic because of this?  From the authoring point of view, most blogging apps don't have any restrictions based on the state.
You can author a draft post in exactly the same way as you author a published post.

The only part of the system that *does* care about the state is the display engine, and that filters out the drafts in the database layer before it ever gets to the domain.

Since there is no special domain logic that cares about the state, it is probably unnecessary.

**State transitions occur outside the application**

In a customer management application, it is common to classify customers as "prospects", "active", "inactive", etc.

![State transition diagram: Customer states](./State_Customer.png)

In the application, these states have business meaning and should be represented by the type system (such as a union type).  But the state *transitions* generally do not occur within the application itself. For example, we might classify a customer as inactive if they haven't ordered anything for 6 months. And then this rule might be applied to customer records in a database by a nightly batch job, or when the customer record is loaded from the database.  But from our application's point of view, the transitions do not happen *within* the application, and so we do not need to create a special state machine.

**Dynamic business rules**

The last bullet point in the list above refers to "static" business rules. By this I mean that the rules change slowly enough that they should be embedded into the code itself.

On the other hand, if the rules are dynamic and change frequently, it is probably not worth going to the trouble of creating static types.

In these situations, you should consider using active patterns, or even a proper rules engine.

## Summary

In this post, we've seen that if you have data structures with explicit flags ("IsVerified") or status fields ("OrderStatus"), or implicit state (clued by an excessive number of nullable or option types), it is worth considering using a simple state machine to model the domain objects.  In most cases the extra complexity is compensated for by the explicit documentation of the states and the elimination of errors due to not handling all possible cases.