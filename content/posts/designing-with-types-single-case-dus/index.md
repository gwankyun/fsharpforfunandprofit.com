---
layout: post
title: "Designing with types: Single case union types"
description: "Adding meaning to primitive types"
date: 2013-01-13
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 2
categories: [Types, DDD]
---

在上一篇文章的末尾，我们有了电子邮件地址、邮政编码等的值，定义如下：

```fsharp

EmailAddress: string;
State: string;
Zip: string;

```

这些都被定义为简单字符串。但说真的，它们只是字符串吗？电子邮件地址可以与邮政编码或州名缩写互换吗？

在领域驱动的设计中，它们确实是不同的东西，而不仅仅是字符串。所以理想情况下，我们希望他们有很多不同的类型，这样他们就不会意外地混淆了。

长期以来，这一直被认为是一种[良好的实践](http://codemonkeyism.com/never-never-never-use-string-in-java-or-at-least-less-often/)，但在C\#和Java等语言中，创建数百个这样的微小类型可能会很痛苦，导致所谓的[「原始痴迷」](http://sourcemaking.com/refactoring/primitive-obsession)代码气味。

但是F\#没有借口！创建简单的包装器类型很简单。

## 包装基本类型 ##

创建单独类型的最简单方法是将基础字符串类型包装在另一个类型中。

我们可以使用单例联合类型来实现，如下所示：

```fsharp
type EmailAddress = EmailAddress of string
type ZipCode = ZipCode of string
type StateCode = StateCode of string
```

或者，我们可以使用一个字段的记录类型，像这样：

```fsharp
type EmailAddress = { EmailAddress: string }
type ZipCode = { ZipCode: string }
type StateCode = { StateCode: string}
```

这两种方法都可以用来创建围绕字符串或其他基本类型的包装器类型，那么哪种方法更好呢？

答案通常是单例联合类型。「wrap」和「unwrap」要容易得多，因为「union case」实际上是一个适当的构造函数。展开可以使用内联模式匹配完成。

下面是一些如何构造和解构`EmailAddress`类型的示例：

```fsharp
type EmailAddress = EmailAddress of string

// 使用构造函数作为函数
"a" |> EmailAddress
["a"; "b"; "c"] |> List.map EmailAddress

// 内联解构
let a' = "a" |> EmailAddress
let (EmailAddress a'') = a'

let addresses =
    ["a"; "b"; "c"]
    |> List.map EmailAddress

let addresses' =
    addresses
    |> List.map (fun (EmailAddress e) -> e)
```

使用记录类型无法轻松做到这一点。

因此，让我们再次重构代码以使用这些联合类型。现在看起来是这样的：

```fsharp
type PersonalName =
    {
    FirstName: string;
    MiddleInitial: string option;
    LastName: string;
    }

type EmailAddress = EmailAddress of string

type EmailContactInfo =
    {
    EmailAddress: EmailAddress;
    IsEmailVerified: bool;
    }

type ZipCode = ZipCode of string
type StateCode = StateCode of string

type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: StateCode;
    Zip: ZipCode;
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

联合类型的另一个优点是可以用模块签名封装实现，我们将在下面讨论。

## 命名单个案例联合的「案例」 ##

在上面的示例中，我们对case使用了与类型相同的名称：

```fsharp
type EmailAddress = EmailAddress of string
type ZipCode = ZipCode of string
type StateCode = StateCode of string
```

这一开始可能会让人感到困惑，但实际上它们位于不同的作用域，因此没有命名冲突。一个是类型，另一个是同名的构造函数。

因此，如果你看到这样的函数签名：

```fsharp
val f: string -> EmailAddress
```

这里指的是类型，所以`EmailAddress`指的是类型。

另一方面，如果你看到这样的代码：

```fsharp
let x = EmailAddress y
```

这里指向值世界中的事物，因此`EmailAddress`指向构造函数。

## 构建单例并集 ##

对于具有特殊含义的值，如电子邮件地址和邮政编码，通常只允许某些值。并不是每个字符串都是可接受的电子邮件或邮政编码。

这意味着我们需要在某些时候进行验证，还有什么比在构建时进行验证更好的呢？毕竟，一旦值被构造出来，它就是不可变的，所以不用担心以后有人会修改它。

下面是我们如何使用一些构造函数扩展上述模块：

```fsharp

... types as above ...

let CreateEmailAddress (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then Some (EmailAddress s)
        else None

let CreateStateCode (s:string) =
    let s' = s.ToUpper()
    let stateCodes = ["AZ";"CA";"NY"] // 等等
    if stateCodes |> List.exists ((=) s')
        then Some (StateCode s')
        else None
```

现在可以测试构造函数了：

```fsharp
CreateStateCode "CA"
CreateStateCode "XX"

CreateEmailAddress "a@example.com"
CreateEmailAddress "example.com"
```

## 处理构造函数中的无效输入 ##

使用这种构造函数，一个直接的挑战是如何处理无效输入。例如，如果我将「abc」传递给电子邮件地址构造函数，会发生什么？

有很多方法可以解决这个问题。

首先，你可以抛出一个异常。我觉得这个又丑又缺乏想象力，所以我马上拒绝了这个！

接下来，你可以返回一个选项类型，`None`表示输入无效。这就是上面的构造函数所做的事情。

这通常是最简单的方法。这样做的好处是调用者必须显式地处理值无效的情况。

例如，上面例子中的调用者代码可能如下所示：

```fsharp
match (CreateEmailAddress "a@example.com") with
| Some email -> ... do something with email
| None -> ... ignore?
```

缺点是使用复杂的验证时，可能不太明显哪里出了问题。邮件是否太长，或者缺少「@」符号，或者域名无效？我们不知道。

如果你确实需要更多细节，你可能想要返回一个包含错误情况下更详细解释的类型。

下面的示例使用`CreationResult`类型在失败情况下指示错误。

```fsharp
type EmailAddress = EmailAddress of string
type CreationResult<'T> = Success of 'T | Error of string

let CreateEmailAddress2 (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then Success (EmailAddress s)
        else Error "Email address must contain an @ sign"

// 测试
CreateEmailAddress2 "example.com"
```

最后，最常用的方法是使用延续。也就是说，传入两个函数，一个用于成功的情况（接受新构造的电子邮件作为参数），另一个用于失败的情况（接受错误字符串作为参数）。

```fsharp
type EmailAddress = EmailAddress of string

let CreateEmailAddressWithContinuations success failure (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then success (EmailAddress s)
        else failure "Email address must contain an @ sign"
```

success函数接受电子邮件作为参数，error函数接受一个字符串参数。这两个函数必须返回相同的类型，但类型由你决定。

这是一个简单的例子——两个函数都执行printf，并且没有返回任何东西（即unit）。

```fsharp
let success (EmailAddress s) = printfn "success creating email %s" s
let failure  msg = printfn "error creating email: %s" msg
CreateEmailAddressWithContinuations success failure "example.com"
CreateEmailAddressWithContinuations success failure "x@example.com"
```

有了延续，你可以轻松地重现任何其他方法。例如，下面是创建选项的方法。在这种情况下，两个函数都返回一个`EmailAddress option`。

```fsharp
let success e = Some e
let failure _  = None
CreateEmailAddressWithContinuations success failure "example.com"
CreateEmailAddressWithContinuations success failure "x@example.com"
```

下面是错误情况下抛出异常的方法：

```fsharp
let success e = e
let failure _  = failwith "bad email address"
CreateEmailAddressWithContinuations success failure "example.com"
CreateEmailAddressWithContinuations success failure "x@example.com"
```

这段代码看起来相当麻烦，但在实践中，你可能会创建一个本地部分应用函数来使用，而不是冗长的函数。

```fsharp
// 设置一个部分应用函数
let success e = Some e
let failure _  = None
let createEmail = CreateEmailAddressWithContinuations success failure

// 使用部分应用函数
createEmail "x@example.com"
createEmail "example.com"
```

{{< book_page_ddd >}}

## 为包装类型创建模块 ##

由于我们添加了验证，这些简单的包装类型开始变得更加复杂，我们可能会发现我们想要与该类型关联的其他函数。

因此，为每个包装类型创建一个模块，并将类型及其相关函数放在其中可能是一个好主意。

```fsharp
module EmailAddress =

    type T = EmailAddress of string

    // 包装
    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then Some (EmailAddress s)
            else None

    // 解包
    let value (EmailAddress e) = e
```

该类型的用户将使用模块函数来创建和解包该类型。例如：

```fsharp

// 创建电子邮件地址
let address1 = EmailAddress.create "x@example.com"
let address2 = EmailAddress.create "example.com"

// 解包电子邮件地址
match address1 with
| Some e -> EmailAddress.value e |> printfn "the value is %s"
| None -> ()
```

## 强制使用构造函数 ##

一个问题是你不能强制调用者使用构造函数。有人可以绕过验证直接创建类型。

在实践中，这往往不是一个问题。一种简单的技术是使用命名约定来指示「私有」类型，并提供「包装」和「解包装」函数，以便客户端永远不需要直接与该类型交互。

这里的一个例子：

```fsharp

module EmailAddress =

    // 私有类型
    type _T = EmailAddress of string

    // 包装
    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then Some (EmailAddress s)
            else None

    // 解包
    let value (EmailAddress e) = e
```

当然，在这种情况下，类型并不是真正的私有，但你鼓励调用者始终使用「已发布」函数。

如果你真的想封装类型的内部并强制调用者使用构造函数，可以使用模块签名。

下面是邮件地址示例的签名文件：

```fsharp
// 文件： EmailAddress.fsi

module EmailAddress

// 封装类型
type T

// 包装
val create : string -> T option

// 解包
val value : T -> string
```

（请注意，模块签名仅在编译项目中有效，在交互式脚本中无效，因此要测试这一点，你需要在F\#项目中创建三个文件，文件名如下所示。）

下面是实现文件：

```fsharp
// 文件： EmailAddress.fs

module EmailAddress

// 封装类型
type T = EmailAddress of string

// 包装
let create (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then Some (EmailAddress s)
        else None

// 解包
let value (EmailAddress e) = e

```

这是一个客户端：

```fsharp
// 文件： EmailAddressClient.fs

module EmailAddressClient

open EmailAddress

// 使用发布的函数时，代码能够正常工作
let address1 = EmailAddress.create "x@example.com"
let address2 = EmailAddress.create "example.com"

// 使用该类型内部的代码无法编译
let address3 = T.EmailAddress "bad email"

```

模块签名导出的`EmailAddress.T`类型是不透明的，因此客户端无法访问内部。

如你所见，这种方法强制使用构造函数。尝试直接创建类型（`T.EmailAddress "bad email"`）会导致编译错误。

## 何时「包装」单例联合 ##

现在我们有了包装类型，我们应该在什么时候构建它们？

通常只需要在服务边界（例如，[六边形体系结构](http://alistair.cockburn.us/Hexagonal+architecture)中的边界）。

在这种方法中，包装在UI层中完成，或者从持久层加载时完成，一旦创建了包装类型，它就会被传递到域层，并作为不透明类型进行「整体」操作。令人惊讶的是，当你在域本身工作时，实际上并不需要直接包装的内容。

作为构造的一部分，调用者使用提供的构造函数而不是执行自己的验证逻辑是至关重要的。这确保了「坏」的值永远不能进入域。

例如，下面的代码显示UI正在进行自己的验证：

```fsharp
let processFormSubmit () =
    let s = uiTextBox.Text
    if (s.Length < 50)
        then // 在域名对象上设置email
        else // 显示验证错误信息
```

更好的方法是让构造函数来完成，如前所述。

```fsharp
let processFormSubmit () =
    let emailOpt = uiTextBox.Text |> EmailAddress.create
    match emailOpt with
    | Some email -> // 在域名对象上设置email
    | None -> // 显示验证错误信息
```

## 何时「解包」单例联合 ##

什么时候需要打开包装？同样，通常只在服务边界。例如，当你将电子邮件持久化到数据库，或绑定到UI元素或视图模型时。

避免显式展开的一个技巧是再次使用延续方法，传入一个将应用于包装值的函数。

也就是说，与其显式调用「解包」函数：

```fsharp
address |> EmailAddress.value |> printfn "the value is %s"
```

你不如传入一个函数，该函数将应用于内部值，如下所示：

```fsharp
address |> EmailAddress.apply (printfn "the value is %s")
```

综上所示，我们现在有了完整的`EmailAddress`模块。

```fsharp
module EmailAddress =

    type _T = EmailAddress of string

    // 使用创建延续
    let createWithCont success failure (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then success (EmailAddress s)
            else failure "Email address must contain an @ sign"

    // 直接创建
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // 使用延续解包
    let apply f (EmailAddress e) = f e

    // 直接解包
    let value e = apply id e

```

`create`和`value`函数并不是严格必需的，添加这两个函数是为了方便调用者。

## 目前的代码 ##

现在让我们重构`Contact`代码，添加新的包装器类型和模块。

```fsharp
module EmailAddress =

    type T = EmailAddress of string

    // 使用创建延续
    let createWithCont success failure (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then success (EmailAddress s)
            else failure "Email address must contain an @ sign"

    // 直接创建
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // 使用延续解包
    let apply f (EmailAddress e) = f e

    // 直接解包
    let value e = apply id e

module ZipCode =

    type T = ZipCode of string

    // 使用创建延续
    let createWithCont success failure  (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\d{5}$")
            then success (ZipCode s)
            else failure "Zip code must be 5 digits"

    // 直接创建
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // 使用延续解包
    let apply f (ZipCode e) = f e

    // 直接解包
    let value e = apply id e

module StateCode =

    type T = StateCode of string

    // 使用创建延续
    let createWithCont success failure  (s:string) =
        let s' = s.ToUpper()
        let stateCodes = ["AZ";"CA";"NY"] //etc
        if stateCodes |> List.exists ((=) s')
            then success (StateCode s')
            else failure "State is not in list"

    // 直接创建
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // 使用延续解包
    let apply f (StateCode e) = f e

    // 直接解包
    let value e = apply id e

type PersonalName =
    {
    FirstName: string;
    MiddleInitial: string option;
    LastName: string;
    }

type EmailContactInfo =
    {
    EmailAddress: EmailAddress.T;
    IsEmailVerified: bool;
    }

type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: StateCode.T;
    Zip: ZipCode.T;
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

顺便说一下，注意我们现在在三个包装类型模块中有相当多的重复代码。有什么好办法可以摆脱它，或者至少让它更干净？

## 总结 ##

总结一下可区分联合的使用，这里有一些指导方针：

* 坚持使用单例可区分联合来创建准确表示域的类型。
* 如果包装的值需要验证，则提供执行验证并强制使用的构造函数。
* 清楚验证失败时会发生什么。在简单的情况下，返回选项类型。在更复杂的情况下，让调用者传递成功和失败的处理程序。
* 如果被包装的值有许多关联的函数，请考虑将其移到单独的模块中。
* 如果你需要强制封装，使用签名文件。

我们还没有完成重构。我们可以改变类型的设计，在编译时强制执行业务规则——使非法状态不可表示。

{{< linktarget "update" >}}

## 更新 ##

很多人都想知道更多关于如何确保约束类型，如`EmailAddress`，只能通过执行验证的特殊构造函数创建的信息。所以我在这里列出了一个[gist here](https://gist.github.com/swlaschin/54cfff886669ccab895a)，里面有一些其他方法的详细例子。
