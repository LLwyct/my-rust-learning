# 10-2 Trait：定义共享行为

trait 告诉 Rust 编译器某个特定类型拥有可能与其他类型共享的功能。可以通过 trait 以一种抽象的方式定义共享的行为。可以使用 trait bounds 指定泛型是任何拥有特定行为的类型。

**注意：trait 类似于其他语言中的常被称为 接口（interfaces）的功能，虽然有一些不同**

## 定义 trait

一个类型的行为由其方法定义，如果不同的类型有着相似的行为，可以使用 trait 实现。

比如，对于传统的新闻文章，我们可以在其上实现一个 **提取摘要** 的方法；对于现代的网络微博，也可以实现一个 **提取摘要** 方法。

```rust
pub trait Summary {
    fn summarize (&self) -> String;
}
```

首先使用 trait 关键字 声明一个trait，后面是 trait名。在大括号声明实现这个trait 的具体方法签名。

不需要在定义 trait 时提供具体的方法体。如果一个类型要实现此 trait ，需要提供自己的方法体，编译器会确保任何实现 `Summary` 的类型都有与这个签名完全一致的 `summarize` 方法。

## 为类型实现 trait

上面说了，定义 trait 时只需要提供方法签名，不需要实现方法体。当具体某个类型要实现该 trait 时，再提供方法体。

比如我们定义了一个 `Summary` 的trait，现在 新闻类型，微博类型 要去实现这个 trait，此时我们实现内部方法 `summarize`。

```ruby
pub trait Summary {
    fn summarize (&self) -> String;
}

pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

pub struct Weibo {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for NewsArticle {
    fn summarize (&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

impl Summary for Weibo {
    fn summarize (&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

上面的代码中，我们分别声明了`Summary` trait、定义了 `NewsArticle`、`Weibo` 结构体、并为这两个类型实现了`Summary` trait。

实现trait 的语法为：`impl xxx for xxxx`。接下来看一下使用：

```ruby
fn main () {
    let weibo = Weibo {
        username: String::from("@Fojuto"),
        content: String::from("今天下雨了"),
        reply: false,
        retweet: false,
    };

    println!("1 new weibo: {}", weibo.summarize());
}
// 1 new weibo: @Fojuto: 今天下雨了
```

假如我们的crate 名为 `demo` 的，而别人想要利用我们 crate 的功能为其自己的库作用域中的结构体实现 `Summary` trait。首先他们需要将 `Summary` 引入作用域。这可以通过指定 `use demo::Summary;` 实现，这样就可以为其类型实现 `Summary` trait 了。`Summary` 还必须是公有 trait 使得其他 crate 可以实现它，这也是为什么我们把 `pub` 置于 `trait` 之前。

> 实现 trait 时需要注意的一个限制是，只有当 trait 或者要实现 trait 的类型位于 crate 的本地作用域时，才能为该类型实现 trait。例如，可以为 aggregator crate 的自定义类型 Tweet 实现如标准库中的 Display trait，这是因为 Tweet 类型位于 aggregator crate 本地的作用域中。类似地，也可以在 aggregator crate 中为 `Vec<T>` 实现 Summary，这是因为 Summary trait 位于 aggregator crate 本地作用域中。
>
> 但是不能为外部类型实现外部 trait。例如，不能在 aggregator crate 中为 `Vec<T>` 实现 Display trait。这是因为 Display 和 `Vec<T>` 都定义于标准库中，它们并不位于 aggregator crate 本地作用域中。这个限制是被称为 **相干性**（coherence） 的程序属性的一部分，或者更具体的说是 **孤儿规则**（orphan rule），其得名于不存在父类型。这条规则确保了其他人编写的代码不会破坏你代码，反之亦然。没有这条规则的话，两个 crate 可以分别对相同类型实现相同的 trait，而 Rust 将无从得知应该使用哪一个实现。

上面哔哔了这么多，用我的话说就是：

自己写的就是内部的，不是自己写的就是外部的。要为 **类型** 实现 **trait** 有限制——两者至少有一个是内部的。

举个例子

* `Summary` 就是一个 trait，`Weibo` 就是一个类型，都是内部的。
* 标准库中的 `Display` 是一个 trait ，标准库中的 `Vec<T>` 是一个类型，都是外部的。

你可以给 `Weibo` 实现 `Display`。也可以给 `Vec<T>`实现 `Summary`。但是不能给`Vec<T>` 实现 `Display`。

## 默认实现

虽说没必要在定义 trait 时实现方法体，但是还是可以给出一个默认实现。（在定义时为其实现，就相当于该方法的默认实现）

```ruby
pub trait Summary {
    fn summarize (&self) -> String {
        String::from("(Read more...)")
    }
}

impl Summary for NewsArticle {} // 用空白块
```

接下来的内容，我认为中文文档会给人造成较大的误解，因此按照我的思路来。

## trait 作为参数类型

有的时候我们会遇到这样一种需求，我希望函数接收的参数它有一个特点——实现了 xxx trait。比如我这个函数只接受实现了 `Summary` 的 类型 的 参数，比如 `NewsArticle` 和 `Weibo`。

### 简单语法 impl trait

这种方式适合简单的例子，比如只有一个参数，它其实是 Trait Bound 的语法糖。

```rust
pub fn notify (item: impl Summary) { ... }
```

notify 这个函数，它接收一个参数 `item`，`item` 必须实现了 `Summary` trait。

### Trait Bound 语法

Bound在这里可以理解为“限制”。

```rust
pub fn notify <T: Summary> (item: T) {...}
```

第三遍看到这里我才反应过来，这不就是函数参数的约束吗？随便拿 typescript 举个例子。

```typescript
interface User {
    id: string;
    username: string;
}
function getUser (user: <T: User>) {
    ...
}
```

一下子就清晰了，怎么用Rust突然巨抽象。。。 其实就是对泛型的类型进行限制嘛。

Trait Bound 语法有什么好处呢？那就是在参数比较多的时候看起来简洁一些。

这是简单语法：

```rust
pub fn notify(item1: impl Summary, item2: impl Summary) {...}
```

这是Trait Bound 语法，是不是简单许多？

```rust
pub fn notify<T: Summary>(item1: T, item2: T) {...}
```

## 通过 + 指定多个Trait Bound

这个很好理解，有时候我们希望一个类型的更多一些，不只由一个 Trait 限制：

```rust
// 简单语法
pub fn notify(item: impl Summary + Display) {...}
// trait bound写法
pub fn notify<T: Summary + Display>(item: T) {...}
```

再用typescript举个例子，我们希望只有管理员和高级VIP会员才能用这个功能，就可以使用这个东西。

## 通过 where 简化 Trait Bound

有时候一个函数的类型情况特别复杂，比如下例：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: T, u: U) -> i32 {...}
```

可以使用 where 从句：

```rust
fn some_function<T, U>(t: T, u: U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug
{...}
```

## 对函数返回值的类型添加 trait 限制

之前是函数的入参有trait 限制，返回值也可以由 trait 限制，只有实现了 `Summary` 的类型才能作为返回值。

```rust
fn returns_summarizable() -> impl Summary {
    Weibo{
        username: String::from("horse_ebooks"),
        content: String::from("of course, as you probably already know, people"),
        reply: false,
        retweet: false,
    }
}
```

> 返回一个只是指定了需要实现的 trait 的类型的能力在闭包和迭代器场景十分的有用，第十三章会介绍它们。闭包和迭代器创建只有编译器知道的类型，或者是非常非常长的类型。`impl Trait` 允许你简单的指定函数返回一个 `Iterator` 而无需写出实际的冗长的类型。

**要极其注意的是**：只适用于返回单一类型的情况。例如，这段代码的返回值类型指定为返回 `impl Summary`，但是返回了 `NewsArticle` 或 `Weibo` 就行不通：

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from("Penguins win the Stanley Cup Championship!"),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from("The Pittsburgh Penguins once again are the best
            hockey team in the NHL."),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from("of course, as you probably already know, people"),
            reply: false,
            retweet: false,
        }
    }
}
```

这里尝试返回 `NewsArticle` 或 `Tweet`。这不能编译，因为 `impl Trait` 工作方式的限制。第十七章的 “[为使用不同类型的值而设计的 trait 对象](https://kaisery.github.io/trpl-zh-cn/ch17-02-trait-objects.html#using-trait-objects-that-allow-for-values-of-different-types)” 部分会介绍如何编写这样一个函数。

## 使用 trait bounds 来修复 largest 函数

### 对于标量的修复方法

还记得示例 10-5 吗？让我们修复一下使用泛型的 `largest` 函数，目前这个函数是无法通过编译的。

```rust
fn largest<T> (list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {}", result);
    let char_list = vec!['y', 'm', 'a', 'q'];
    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

看一下报错信息：

```text
binary operation `>` cannot be applied to type `T`
```

我们在这里想要比较两个 T 类型的值的大小，但是众所周知，一般只有基础类型才能直接比大小。事实上，在底层实现 `>` 运算符的是标准库中 trait `std::cmp::PartialOrd` 的一个默认方法。因此，只要给 `T` 加上 `partialOrd` 的 限制即可。`partialOrd` 为 preclude。

> 这里再加强一下理解，上面的内容可能比较抽象。bound 这个词在Rust 的语境中不翻译为边界，翻译为限制，trait bound 意为 trait 限制，trait 直译为特质、特征，与其他语言中的接口 interface 等价，类型的 trait bound 理解为，为类型添加限制——该类型必须实现了该trait 中的某些功能。

因此只需要稍微改动一下第一行代码：

```rust
fn largest<T: PartialOrd> (list: &[T]) -> T {
```

但是又出现了新的错误：`let mut largest = list[0];` 提示 `cannot move out of type [T], a non-copy slice`。这里的list是切片类型，是引用类型，没有所有权，但是list\[0\]，是 T 类型，之前讲到，如果一个没有实现 Copy trait的类型，发生赋值操作，会发生 **移动**，所有权会转让，从而使 list\[0\] 失效。编译器认为这里的类型有可能没有实现 Copy trait，因此不允许进行 移动。所以，我们只需要再添加 trait bound Copy即可。

```rust
fn largest<T: PartialOrd + Copy> (list: &[T]) -> T {
```

### 对于复合变量的修复方法

这也是因为我们事先知道，此处的容器元素类型为已实现了Copy trait的 i32、char 等标量数据类型，如果为复合数据类型，比如String，还需要再改进。我们可以指定 trait bound 为 Clone 而不是Copy。并克隆 slice 的每一个值使得 largest 函数拥有其所有权。使用 clone 函数意味着对于类似 String 这样拥有堆上数据的类型，会潜在的分配更多堆上空间，而堆分配在涉及大量数据时可能会相当缓慢。

String 类型版本如下：

```rust
fn largest<T: PartialOrd + Clone> (list: &[T]) -> T {
    let mut largest = list[0].clone(); // 这里使用 clone
    for item in list.iter() { // 这里相当于发生了list.iter移动到了item，不允许，因此定义item为引用
        if item > &largest { // item 为引用类型，largest为实体类型，要给largest加引用
            largest = item.clone();
        }
    }
    largest
}
```

### 究极修复方法

```rust
fn largest<T: PartialOrd + Clone> (list: &[T]) -> &T { // 令返回值为引用
    let mut largest = &list[0];
    for item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

## trait 作为impl参数类型限制

之前学过，可以为结构体等使用 `impl` 关键字实现方法。其实在用 impl 实现的时候，也可以使用 trait 添加限制条件。

比如在下面的例子中

* 为所有 Pair 类型添加new方法
* 只为 Pair 中存放的 实现了 `Display` + `PartialOrd` 的 数据类型添加 `cmp_display` 方法。

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self {
            x,
            y,
        }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

对任何满足特定 trait bound 的类型实现 trait 被称为 _blanket implementations_，他们被广泛的用于 Rust 标准库中。例如，标准库为任何实现了 Display trait 的类型实现了 ToString trait。这个 impl 块看起来像这样：

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

因为标准库有了这些 blanket implementation，我们可以对任何实现了 `Display` trait 的类型调用由 ToString 定义的 `to_string` 方法。例如，可以将整型转换为对应的 String 值，因为整型实现了 Display：

```rust
let s = 3.to_string();
```

blanket implementation 会出现在 trait 文档的 “Implementers” 部分。

trait 和 trait bound 让我们使用泛型类型参数来减少重复，并仍然能够向编译器明确指定泛型类型需要拥有哪些行为。因为我们向编译器提供了 trait bound 信息，它就可以检查代码中所用到的具体类型是否提供了正确的行为。在动态类型语言中，如果我们尝试调用一个类型并没有实现的方法，会在运行时出现错误。Rust 将这些错误移动到了编译时，甚至在代码能够运行之前就强迫我们修复错误。另外，我们也无需编写运行时检查行为的代码，因为在编译时就已经检查过了，这样相比其他那些不愿放弃泛型灵活性的语言有更好的性能。

这里还有一种泛型，我们一直在使用它甚至都没有察觉它的存在，这就是 **生命周期**（lifetimes）。不同于其他泛型帮助我们确保类型拥有期望的行为，生命周期则有助于确保引用在我们需要他们的时候一直有效。让我们学习生命周期是如何做到这些的。

