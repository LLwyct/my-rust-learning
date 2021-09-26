# 6-1 枚举

本章介绍 **枚举**（enumerations），也被称作 enums。枚举允许你通过列举可能的 成员（variants） 来定义一个类型。首先，我们会定义并使用一个枚举来展示它是如何连同数据一起编码信息的。接下来，我们会探索一个特别有用的枚举，叫做 `Option`，它代表一个值要么是某个值要么什么都不是。然后会讲到在 `match` 表达式中用模式匹配，针对不同的枚举值编写相应要执行的代码。最后会介绍 `if let`，另一个简洁方便处理代码中枚举的结构。

枚举是一个很多语言都有的功能，不过不同语言中其功能各不相同。Rust 的枚举与 F\#、OCaml 和 Haskell 这样的函数式编程语言中的 代数数据类型（algebraic data types）最为相似。

## 定义枚举

加入现在要处理IP地址，给出以下枚举类型：

```typescript
enum IpAddrKind {
    V4,
    V6
}
```

其中 `V4，V6` 称为成员，现在 `IpAddrKind` 就是一个可以在代码中使用的自定义数据类型了。

## 枚举值

```typescript
enum IpAddrKind {
    V4,
    V6
}
struct IpAddr {
    kind: IpAddrKind,
    address: String
}
fn main() {
    let ipv4 = IpAddrKind::V4;
    let ip = IpAddr {
        kind: IpAddrKind::V4,
        address: String::from("192.168.0.1")
    };
}
```

还有一种更简洁的方式来表达相同的概念，使用枚举并将数据直接放进每一个枚举成员而不是将枚举作为结构体的一部分。 `IpAddr` 枚举的新定义表明了 `V4` 和 `V6` 成员都关联了 `String` 值。

```typescript
enum IpAddr {
    V4(String),
    V6(String)
}
fn main() {
    let home = IpAddr::V4(String::from("127.0.0.1"));
}
```

我们直接将数据附加到枚举的每个成员上，这样就不需要一个额外的结构体了。

用枚举替代结构体还有另一个优势：每个成员可以处理不同类型和数量的数据。IPv4版本的 IP 地址总是含有四个值在 0 和 255 之间的数字部分。如果我们想要将 `V4` 地址存储为四个 `u8` 值而 `V6` 地址仍然表现为一个 `String`，这就不能使用结构体了。枚举则可以轻易的处理这个情况：

```typescript
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String)
}
let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String.from("::1"));
```

### 一个复杂的例子：枚举与结构体

```typescript
示例 6-2
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

这个枚举有四个含有不同类型的成员：

* `Quit` 没有关联任何数据。
* `Move` 包含一个匿名结构体。
* `Write` 包含单独一个 String。
* `ChangeColor` 包含三个 i32。

上述这种定义枚举的方式和定义多个不同类型的结构体的方式很像，除了枚举不使用 `struct` 关键字以及其所有成员都被组合于 `Message` 类型下。如下这些结构体可以包含与之前枚举成员中相同的数据，效果相同：

```ruby
struct QuitMessage; // 类单元结构体
struct MoveMessage {
    x: i32,
    y: i32,
}
struct WriteMessage(String); // 元组结构体
struct ChangeColorMessage(i32, i32, i32); // 元组结构体
```

不过，如果我们使用不同的结构体，由于它们都有不同的类型，我们将不能像使用示例 6-2 中定义的 Message 枚举那样，轻易的定义一个能够处理这些不同类型的结构体的函数，因为枚举是单独一个类型。

结构体和枚举还有另一个相似点：就像可以使用 `impl` 来为结构体定义方法那样，也可以在枚举上定义方法。这是一个定义于我们 `Message` 枚举上的叫做 `call` 的方法：

```ruby
impl Message {
    fn call (&self) {/* TODO */}
}
let m = Message::Write(String::from("hello"));
m.call();
```

方法体使用了 `self` 来获取调用方法的值。这个例子中，创建了一个值为 `Message::Write(String::from("hello"))` 的变量 `m`，而且这就是当 `m.call()` 运行时 `call` 方法中的 `self` 的值。

让我们看看标准库中的另一个非常常见且实用的枚举：`Option`。

## Option 枚举和其相对于空值的优势

编程语言的设计经常要考虑包含哪些功能，但考虑排除哪些功能也很重要。Rust 并没有很多其他语言中有的空值功能。**空值（Null ）**是一个值，它代表没有值。**在有空值的语言中，变量总是这两种状态之一：空值和非空值。**

空值的问题在于当你尝试像一个非空值那样使用一个空值，会出现某种形式的错误。因为空和非空的属性无处不在，非常容易出现这类错误。

然而，空值尝试表达的概念仍然是有意义的：空值是一个因为某种原因目前无效或缺失的值。

问题不在于概念而在于具体的实现。为此，Rust 并没有空值，不过它确实拥有一个可以编码存在或不存在概念的枚举。这个枚举是 `Option<T>`，而且它定义于标准库中，如下:

```typescript
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` 枚举是如此有用以至于它甚至被包含在了 prelude 之中，**你不需要将其显式引入作用域**。另外，它的成员也是如此，**可以不需要 `Option::` 前缀来直接使用 `Some` 和 `None`**。即便如此 `Option<T>` 也仍是常规的枚举，`Some(T)` 和 `None` 仍是 `Option<T>` 的成员。

`<T>` 语法是一个我们还未讲到的 Rust 功能。它是一个泛型类型参数，第十章会更详细的讲解泛型。目前，所有你需要知道的就是 `<T>` 意味着 Option 枚举的 `Some` 成员可以包含任意类型的数据。这里是一些包含数字类型和字符串类型 `Option` 值的例子：

```typescript
let some_number = Some(5);
let some_string = Some("a string");
let absent_number: Option<i32> = None;
```

如果使用 `None` 而不是 `Some`，需要告诉 Rust `Option<T>` 是什么类型的，因为编译器只通过 `None` 值无法推断出 `Some` 成员保存的值的类型。

当有一个 Some 值时，我们就知道存在一个值，而这个值保存在 Some 中。当有个 None 值时，在某种意义上，它跟空值具有相同的意义：并没有一个有效的值。那么，`Option<T>` 为什么就比空值要好呢？

简而言之，因为 `Option<T>` 和 `T`（这里 `T` 可以是任何类型）是不同的类型，编译器不允许像一个肯定有效的值那样使用 `Option<T>`。例如，这段代码不能编译，因为它尝试将 `Option<i8>` 与 `i8` 相加：

```typescript
let x: i8 = 5;
let y: Option<i8> = Some(5);
let sum = x + y;
no implementation for `i8 + std::option::Option<i8>`
```

很好！事实上，错误信息意味着 Rust 不知道该如何将 `Option<i8>` 与 `i8` 相加，因为它们的类型不同。当在 Rust 中拥有一个像 `i8` 这样类型的值时，编译器确保它总是有一个有效的值。我们可以自信使用而无需做空值检查。只有当使用 `Option<i8>`（或者任何用到的类型）的时候需要担心可能没有值，而编译器会确保我们在使用值之前处理了为空的情况。

换句话说，在对 `Option<T>` 进行 `T` 的运算之前必须将其转换为 `T`。通常这能帮助我们捕获到空值最常见的问题之一：假设某值不为空但实际上为空的情况。

不再担心会错误的假设一个非空值，会让你对代码更加有信心。为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 `Option<T>` 中。接着，当使用这个值时，必须明确的处理值为空的情况。只要一个值不是 `Option<T>` 类型，你就 可以 安全的认定它的值不为空。这是 Rust 的一个经过深思熟虑的设计决策，来限制空值的泛滥以增加 Rust 代码的安全性。

那么当有一个 `Option<T>` 的值时，如何从 `Some` 成员中取出 `T` 的值来使用它呢？`Option<T>` 枚举拥有大量用于各种情况的方法：你可以查看[它的文档](https://doc.rust-lang.org/std/option/enum.Option.html)。熟悉 `Option<T>` 的方法将对你的 Rust 之旅非常有用。

总的来说，为了使用 `Option<T>` 值，需要编写处理每个成员的代码。你想要一些代码只当拥有 `Some(T)` 值时运行，允许这些代码使用其中的 `T`。也希望一些代码在值为 `None` 时运行，这些代码并没有一个可用的 `T` 值。`match` 表达式就是这么一个处理枚举的控制流结构：它会根据枚举的成员运行不同的代码，这些代码可以使用匹配到的值中的数据。

