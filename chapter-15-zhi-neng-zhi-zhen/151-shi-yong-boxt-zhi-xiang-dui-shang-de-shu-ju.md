# 15-1 使用Box&lt;T&gt;指向堆上的数据

## Box&lt;T&gt;

* 这是最简单的智能指针
* 允许你在heap 上存放数据，stack 上的是指向heap的指针。
* 没有性能开销
* 没有其他额外功能

**使用场景**：

1. 当有一个在编译时未知大小的类型，而又想要在需要确切大小的上下文中使用这个类型值的时候
2. 当有大量数据并希望在确保数据不被拷贝的情况下转移所有权的时候
3. 当希望拥有一个值并只关心它的类型是否实现了特定 trait 而不是其具体类型的时候

> 我们会在 “box 允许创建递归类型” 部分展示第一种场景。在第二种情况中，转移大量数据的所有权可能会花费很长的时间，因为数据在栈上进行了拷贝。为了改善这种情况下的性能，可以通过 box 将这些数据储存在堆上。接着，只有少量的指针数据在栈上被拷贝。第三种情况被称为 trait 对象（trait object），第十七章刚好有一整个部分 “为使用不同类型的值而设计的 trait 对象” 专门讲解这个主题。所以这里所学的内容会在第十七章再次用上！

## 使用 `Box<T>` 在堆上存储数据

```rust
fn main (){
    let b = Box::new(5);
    println!("b={}", b);
}
```

众所周知像 `i32` 这种标量类型是可以直接放在栈上的，但是如果我们用 `Box` 进行封装，就可以把 `5` 对应的值存放在 `heap`上，而`b` 本身则被存放在 `stack` 上，且 `b` 拥有 `5` 的所有权，当 `b` 的到达花括号的时候，`b` 以及 `5` 都会被随之销毁。

而且有趣的是，这里可以直接使用 `b` 来访问数据。

不过将一个单独的值存放在堆上没有意义，上例这样单独使用 `Box` 不常见。将像单个 `i32` 这样的值储存在栈上，也就是其默认存放的地方在大部分使用场景中更为合适。

**所以让我们看看一个不使用 `Box` 时无法定义的类型的例子**。

## Box 允许创建递归类型

介绍一个RUST知识点：**Rust 需要在编译时知道类型占用多少空间。**

因此对于递归类型（其部分值的类型是它本身），RUST不知道需要多少空间。

比如在JS中定义一个树结构，其左右子树其实是自身的类型，这在RUST中是无法直接定义的。

不过 `Box` 有已知大小，因此在递归类型中使用 Box 代替递归的那一部分就可以创建递归类型了。

（我回想起了C语言学习链表的时候，尾指针往往是自身的指针类型，是否和这个是一个道理？）

## Cons List

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2911a7bb3c3f41c79a77753b60199934.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_8,color_FFFFFF,t_70,g_se,x_16)

上面提到的这种递归数据结构在 Lisp 这个语言中称为 Cons List。

Cons List 里每个成员由两个元素组成

* 当前项的值
* 下一个元素

当下一个元素的值为 Nil 时表示结束。

但是 Cons List 在 Rust 中不常用，因为Rust 有非常好用的 迭代器`Vec<T>` 。

### 在RUST中实现Cons List

```rust
use crate::List::{Cons, Nil};
fn main () {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}

enum List {
    Cons(i32, List),
    Nil,
}
```

这里编译器直接提示

```text
error: recursive type `List` has infinite size
```

对于枚举类型，Rust 取最大的元素空间大小为本身的大小。

对此，编译器也做出了提示：这里不应该直接存储数据，而是应该存储一个指针，因为指针的大小是确定的。

### 使用`Box<T>`来获得大小确定的递归类型

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/258879b1c0414bec964b39e9cdf00cf4.png)

```rust
use crate::List::{Cons, Nil};
fn main () {
    let list = Cons(
        1,
        Box::new(Cons(
            2,
            Box::new(Cons(
                3,
                Box::new(Nil)
            ))
        ))
    );
}

enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

## 总结 Box

`Box<T>` 是一个指针，Rust 知道它需要多少空间。

* 只提供了 间接存储 和 heap 内存分配的功能
* 没有其他额外功能
* 没有性能开销
* 适用于需要间接存储的场景，例如 Cons List
* 实现了 `Deref` trait 和 `Drop` trait

`Box<T>` 类型是一个智能指针，因为它实现了 `Deref` trait，它允许 `Box<T>` 值被当作引用对待。当 `Box<T>` 值离开作用域时，由于 `Box<T>` 类型 `Drop` trait 的实现，box 所指向的堆数据也会被清除。让我们更详细的探索一下这两个 trait。这两个 trait 对于在本章余下讨论的其他智能指针所提供的功能中，将会更为重要。

