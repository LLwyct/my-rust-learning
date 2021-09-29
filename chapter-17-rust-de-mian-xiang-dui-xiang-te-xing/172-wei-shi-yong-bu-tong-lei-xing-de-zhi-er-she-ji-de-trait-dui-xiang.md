# 17-2 为使用不同类型的值而设计的 trait 对象

现在有个场景，我要编写一个GUI库，首先需要一个screen类，用于收纳组件，其中有许多组件components，只需要生成组件实例，并挂载到screen上，最后遍历screen中的组件，并调用其上的 `draw` 方法，就可以绘制到屏幕上。

我作为库的开发者，我只实现`Screen`类，并且实现一个`Button Component`。使用我们的库的人，可以实现它自己的组件，只需要保证为他的库实现`draw`方法即可。

**思考传统面向对象语言如何实现？**

首先，定义一个父类，`Component`，类上有 `draw` 方法，其他子类，比如 `Button`，`Image`等组件需要继承自`Component`类，并覆盖实现自己的draw方法。

不过，Rust并没有自己的继承，需要另寻出路！

## 定义通用行为的Trait

**定义 Draw Trait**

为了实现组件期望的通用行为`draw`，我定义一个 Draw Trait ，其中包含 `draw`方法。

```rust
// lib.rs
pub trait Draw {
    fn draw (&self);
}
```

## 创建 Screen 结构体

接着创建`Screen` struct，并且使它拥有一个存放 **Trait 对象**的容器。

**Trait 对象**是指实现了该Trait 的结构体实例（也可以是其他数据结构）。

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}
impl Screen {
    pub fn run (&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

### 为什么使用鸭子类型？

注意这里容器中 `components` 的类型，`Box<dyn Draw>`，Box可以理解，表示智能指针，`dyn Draw` 是指所有实现了Draw Trait的类型，即Trait对象。

**什么是鸭子类型？**

只关心值所反映的信息而不是其具体类型：如果它走起来像一只鸭子，叫起来像一只鸭子，那么它就是一只鸭子！

在此处，我们不关心具体的类型，只关心这个类型有没有实现Draw Trait。

### 为什么不使用泛型？要进行动态分发？

第十章：[泛型代码的性能](https://kaisery.github.io/trpl-zh-cn/ch10-01-syntax.html#performance-of-code-using-generics) 一节讲到，对泛型进行 trait bound时，编译器会进行单态化处理，即在编译时就能推断出 T 的具体类型，并且固定。单态化所产生的的代码会被静态分发。

而此处，使用 trait 对象时，必须使用动态分发，因为编译器无法知晓 trait 对象的类型，这是一个鸭子类型，其真实的类型可能是多种多样的。

这一次操作，会带来性能上的退化，但为了额外的灵活性，仍然需要权衡取舍。

## 创建 Button 组件

```rust
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // -- snip --
    }
}
```

## 库的使用者创建自己的代码

```rust
// main.rs
use gui::Draw;
use gui::{Button, Screen};

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // -- snip --
    }
}

fn main() {
    let btn = Button {
        width: 10,
        height: 5,
        label: String::from("Ok"),
    };

    let sbox = SelectBox {
        width: 20,
        height: 10,
        options: vec![String::from("male"), String::from("female")],
    };
    let components: Vec<Box<dyn Draw>> = vec![Box::new(btn), Box::new(sbox)];
    let screen = Screen {
        components,
    };
    screen.run();
}
```

## Trait 对象要求对象安全

只有 **对象安全**（object safe）的 trait 才可以组成 trait 对象。使得 trait 对象安全的属性存在复杂的规则，不过在实践中，只涉及到两条。如果一个 trait 中所有的方法有如下属性时，则该 trait 是**对象安全**的：

* 返回值类型不为 `Self`
* 方法没有任何泛型类型参数

一个不是对象安全的例子是标准库中的 `Clone` trait。`Clone` trait 的 `clone` 方法的参数签名看起来像这样：

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}

pub struct Screen {
    pub components: Vec<Box<dyn Clone>>,
}
```

将会得到如下错误：

```text
error[E0038]: the trait `std::clone::Clone` cannot be made into an object
 --> src/lib.rs:2:5
  |
2 |     pub components: Vec<Box<dyn Clone>>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `std::clone::Clone`
  cannot be made into an object
  |
  = note: the trait cannot require that `Self : Sized`
```

如果你对**对象安全**的更多细节感兴趣，请查看 [Rust RFC 255](https://github.com/rust-lang/rfcs/blob/master/text/0255-object-safety.md)。

