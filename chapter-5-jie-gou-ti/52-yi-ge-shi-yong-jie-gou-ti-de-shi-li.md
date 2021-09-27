# 5-2 一个使用结构体的示例

## 一个demo

我们现在利用结构体来实现一个计算长方形面积的代码。

我决定搞的面向对象一些，因此在这里使用结构体。

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {

    let rect1 = Rectangle {
        width: 43,
        height: 53
    }; // 不可变借用

    println!("area of rect is {}", area(&rect1));
}

fn area(rect: &Rectangle) -> u32{
    rect.width * rect.height
}
```

## 通过派生 trait 增加实用功能

有一些语言比如JS、Go可以直接打印一些复合类型变量，十分方便，试一试Rust可不可以。

```rust
println!("area of rect is {}\n{}", area(&rect1), rect1);
```

```text
error[E0277]: `Rectangle` doesn't implement `std::fmt::Display`
`Rectangle` cannot be formatted with the default formatter

= help: the trait `std::fmt::Display` is not implemented for `Rectangle`
= note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
```

可以看到这里给出了两条报错和建议。

* 没有为Rectangle实现 `std::fmt::Dispaly` 这个 trait
* 在格式化字符串中可以使用 `{:?} or {:#?}` 替代

所以改进一下代码

```rust
println!("area of rect is {}\n{:#?}", area(&rect1), rect1);
```

```text
= help: the trait `Debug` is not implemented for `Rectangle`
= note: add `#[derive(Debug)]` or manually implement `Debug
```

然后又提示，添加 `#[derive(Debug)]` 或手动实现Debug。

我们这里在结构体上面添加这行代码。

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}
```

最后完美运行

```rust
area of rect is 2279
Rectangle {
    width: 43,
    height: 53,
}
```

> Rust 为我们提供了很多可以通过 derive 注解来使用的 trait，他们可以为我们的自定义类型增加实用的行为。附录 C 中列出了这些 trait 和行为。第十章会介绍如何通过自定义行为来实现这些 trait，同时还有如何创建你自己的 trait。

我们的 area 函数是非常特殊的，它只计算长方形的面积。如果这个行为与 Rectangle 结构体再结合得更紧密一些就更好了，因为它不能用于其他类型。现在让我们看看如何继续重构这些代码，来将 area 函数协调进 Rectangle 类型定义的 area 方法 中。

