# 15-6 引用循环与内存泄露

Rust 的内存安全性会尽可能的保证内存使用安全，但偶尔也会发生意外。Rust 并不能完美地保证可以避免 **内存泄漏** 这意味着**内存泄漏在 Rust 被认为是内存安全的**。

通过 `Rc<T>` 和 `RefCell<T>` 看出：创建引用循环的可能性是存在的。这会造成内存泄漏，因为每一项的引用计数永远也到不了 0，其值也永远不会被丢弃。

## 制造引用循环

```rust
use std::{rc::Rc};
use std::cell::RefCell;
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil
}

impl List {
    fn tail (&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None
        }
    }
}

fn main () {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));
    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));
    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing a = {}", Rc::strong_count(&a));
    // println!("a next item = {:?}", a.tail());
}
```

如果保持最后的 `println!` 行注释并运行代码，会得到如下输出：

```text
a initial rc count = 1
a next item = Some(RefCell { value: Nil })
a rc count after b creation = 2
b initial rc count = 1
b next item = Some(RefCell { value: Cons(5, RefCell { value: Nil }) })
b rc count after changing a = 2
a rc count after changing a = 2
可以看到将 a 修改为指向 b 之后，a 和 b 中都有的 Rc<List> 实例的引用计数为 2。在 main 的结尾，Rust 会尝试首先丢弃 b，这会使 a 和 b 中 Rc<List> 实例的引用计数减 1。
```

然而，因为 a 仍然引用 b 中的 `Rc<List>`，`Rc<List>` 的引用计数是 1 而不是 0，所以 `Rc<List>` 在堆上的内存不会被丢弃。其内存会因为引用计数为 1 而永远停留。为了更形象的展示，我们创建了一个如图所示的引用循环：

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2f2e66f45ffc4ae799f2b9bb050977ee.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_20,color_FFFFFF,t_70,g_se,x_16)

如果取消最后 `println!` 的注释并运行程序，Rust 会尝试打印出 a 指向 b 指向 a 这样的循环直到栈溢出。

创建引用循环并不容易，但也不是不可能。**如果你有包含 `Rc<T>` 的 `RefCell<T>` 值或类似的嵌套结合了内部可变性和引用计数的类型，请务必小心确保你没有形成一个引用循环**；你无法指望 Rust 帮你捕获它们。创建引用循环是一个程序上的逻辑 bug，你应该使用自动化测试、代码评审和其他软件开发最佳实践来使其最小化。

另一个解决方案是重新组织数据结构，使得一部分引用拥有所有权而另一部分没有。换句话说，循环将由一些拥有所有权的关系和一些无所有权的关系组成，只有所有权关系才能影响值是否可以被丢弃。

## 避免引用循环：将 `Rc<T>` 变为 `Weak<T>`

 到目前为止，我们已经展示了调用 `Rc::clone` 会增加 `Rc<T>` 实例的 `strong_count`，和只在其 `strong_count` 为 0 时才会被清理的 `Rc<T>` 实例。你也可以通过调用 `Rc::downgrade` 并传递 `Rc<T>` 实例的引用来创建其值的 **弱引用**（_weak reference_）。调用 `Rc::downgrade` 时会得到 `Weak<T>` 类型的智能指针。不同于将 `Rc<T>` 实例的 `strong_count` 加1，调用 `Rc::downgrade` 会将 `weak_count` 加1。`Rc<T>` 类型使用 `weak_count` 来记录其存在多少个 `Weak<T>` 引用，类似于 `strong_count`。其区别在于 `weak_count` 无需计数为 0 就能使 `Rc<T>` 实例被清理。

略，详情请看[官方文档](https://kaisery.github.io/trpl-zh-cn/ch15-06-reference-cycles.html#%E9%81%BF%E5%85%8D%E5%BC%95%E7%94%A8%E5%BE%AA%E7%8E%AF%E5%B0%86-rct-%E5%8F%98%E4%B8%BA-weakt)

## 总结

这一章涵盖了如何使用智能指针来做出不同于 Rust 常规引用默认所提供的保证与取舍。`Box<T>` 有一个已知的大小并指向分配在堆上的数据。`Rc<T>` 记录了堆上数据的引用数量以便可以拥有多个所有者。`RefCell<T>` 和其内部可变性提供了一个可以用于当需要不可变类型但是需要改变其内部值能力的类型，并在运行时而不是编译时检查借用规则。

我们还介绍了提供了很多智能指针功能的 trait `Deref` 和 `Drop`。同时探索了会造成内存泄漏的引用循环，以及如何使用 `Weak<T>` 来避免它们。

如果本章内容引起了你的兴趣并希望现在就实现你自己的智能指针的话，请阅读 [“The Rustonomicon”](https://doc.rust-lang.org/stable/nomicon/) 来获取更多有用的信息。

接下来，让我们谈谈 Rust 的并发。届时甚至还会学习到一些新的对并发有帮助的智能指针。

