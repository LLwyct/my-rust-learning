# 15-5 RefCell&lt;T&gt;与内部可变性模式

## 什么是内部可变性？

内部可变性（Interior mutability）是Rust中的一种设计模式，他允许你即使在有不可变引用时也可改变数据，这通常是借用规则不允许的。

> 该模式内部使用了 `unsafe` 代码来模糊可变性和借用规则，不展开。

也就是说，当我们人为可以确定代码的正确性时，可以让编译器强制通过那些它认为不正确的代码。

在不可变值内部改变值就是 **内部可变性** 模式。让我们看看何时内部可变性是有用的，并讨论这是如何成为可能的。

## 内部可变性：不可变值的可变借用

借用规则的一个推论是**当有一个不可变值时，不能可变地借用它**。例如，如下代码不能编译：

```rust
fn main() {
    let x = 5; // consider changing this to `mut x`
    let y = &mut x; // cannot borrow mutably
}
// cannot borrow immutable local variable `x` as mutable
```

然而，特定情况下，令一个值在其方法内部能够修改自身，而在其他代码中仍视为不可变，是很有用的。值方法外部的代码就不能修改其值了。`RefCell<T>` 是一个获得内部可变性的方法。`RefCell<T>` 并没有完全绕开借用规则，编译器中的借用检查器允许内部可变性并相应地在运行时检查借用规则。如果违反了这些规则，会出现 panic 而不是编译错误。

让我们在下一个段落通过一个实际的例子来探索何处可以使用 `RefCell<T>` 来修改不可变值并看看为何这么做是有意义的。

## 一个内部可变性的例子：mock 对象

这一部分不论是在 官方文档中，还是视频资料中，都介绍的十分抽象，因为学习者们不懂这个东西的前因后果，不懂为什么要用这么个东西，我在这里就讲一下我自己的理解。

在官方文档中，举了这么一个例子：mock 对象。

假设我们现在要实现一个论坛系统，当发生一些特殊事件的时，比如特别关注的人发了新动态，又或是订阅的服务即将到期等。如果有必要，需要向用户直接发送邮件，甚至手机短信进行通知。因为大多情况下，用户很难一直在网页端守着。

但是我们在开发的时候是分模块开发的，现在分为两个模块：

* **基础功能模块**，发帖，关注，点赞，收藏等。
* **订阅系统模块**，收到私信可以短信/邮件通知，特别关注的人发了新动态可以短信/邮件通知等。

当论坛**基础功能模块**搭建好了，可以知道特别关注发了新的动态，但是要把这一事件发送给用户的**订阅系统模块**还没有做好。

难道基础功能的开发人员要一直等订阅模块开发完才进行下一步的开发吗？当然不行。于是基础功能的开发人员，写了一个 mock 模块，又称“假模块”，先假设这一部分已经完成，来进行测试。

上面说了这么多，就是今天的这个例子的上下文，你知道了这些，应该可以理解为什么要这样写下面的代码了。

接下来，我带你一步步理解这个代码写作的思路。下面的代码都在 `lib.rs` 中完成。

首先，基础模块开发人员就想，我只需要按照规则对事件进行打包，发送给订阅系统，订阅系统怎么处理这个事件与我无瓜，于是就假设这个订阅系统有一个send函数，我只需要把事件通过订阅系统的`send`函数发给自己就可以了，后续要进一步操作我也管不着。

于是先编写一个 Messenger Trait，表示这个订阅类，需要实现这样的一个方法。

```rust
pub trait MessengerTrait {
    fn send (&self, msg: &str);
}
```

然后先不管这个 Messenger 怎么实现，先写自己的部分。

我自己需要根据一定的逻辑判断是否需要发送消息到订阅系统，于是我先编写一个类，类里要包含一个 订阅类 的实例，这样就可以直接在自身调用 `send` 方法。但是我们并不清楚，最终**订阅系统模块**的人要给他们的模块类起什么名字，所以我这里先使用泛型代替，并且这个泛型要实现 `send` 方法。

```rust
pub struct LimitTracker<'a, T: MessengerTrait> {
    messenger: &'a T,
    value: usize,
    max: usize,
}
```

并且我们的发送订阅的依据是（这里是胡编的，毕竟只是个例子）：当 value / max 的值超过一个阈值，就代表这个事很重要需要发送。**当然我这里为了方便就没有逻辑了，随便写一下**。

```rust
impl<'a, T> LimitTracker<'a, T> 
    where T: MessengerTrait
{
    pub fn new (messenger: &T, max: usize) -> LimitTracker<T>{
        LimitTracker {
            messenger,
            max,
            value: 0,
        }
    }

    pub fn set_value (&mut self, value: usize){
        // 这里没有逻辑，直接发
        self.value = value;
        self.messenger.send(&(format!("now value is {}", value))[..]);
    }
}
```

OK，到了这一步，基础模块功能就编写完了，现在编写**订阅系统模块**。

在**订阅系统模块**，我们当然要实现一个“假模块” `MockMessenger`。并且实现`MessengerTrait` 中的 `send` 方法，很简单，就是把一句话 push 进它的 `msg_pool` 队列中，推进去以后，就不关我们的事了。

```rust
mod tests {
    use super::*;

    struct MockMessenger {
        msg_pool: Vec<String>,
    }

    impl MockMessenger {
        fn new () -> MockMessenger{
            MockMessenger {
                msg_pool: Vec::new(),
            }
        }
    }

    impl MessengerTrait for MockMessenger {
        fn send(&self, msg: &str) {
            self.msg_pool.push(String::from(msg));
        }
    }
    ...
```

再编写一下测试代码，新建一个 `MockMessenger` 对象，再新建一个 `LimitTracker` 对象，把前者的引用，和 `max=100` 作为参数传入后者的构造函数，然后调用前者的 `set_value` 即可。

那么 订阅类 `messenger` 就应该有一条消息 `"now value is 20"`。

```rust
    ...
    #[test]
    fn func () {
        let messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&messenger, 100);
        limit_tracker.set_value(20);
        assert_eq!(messenger.msg_pool[0], "now value is 20");
    }
}
```

代码写到这基本就完成了，但是编译无法通过。大意是，不能修改 `MockMessenger` 来记录消息，因为 `send` 方法获取了 `self` 的不可变引用。我们也不能参考错误文本的建议使用 `&mut self` 替代，因为这样 send 的签名就不符合 `MessengerTrait` trait 定义中的签名了（可以试着这么改，看看会出现什么错误信息）。

```text
error[E0596]: cannot borrow `self.msg_pool` as mutable, as it is behind a `&` reference
  --> src\lib.rs:45:13
   |
2  |     fn send (&self, msg: &str);
   |              ----- help: consider changing that to be a mutable reference: `&mut self`
...
45 |             self.msg_pool.push(String::from(msg));
   |             ^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable
```

主要是在 `send` 函数，调用了自身的不可变引用，然后在不可变引用上改变自身成员 `msg_pool` 的内容。这是不允许的，但是最重要的其实是在这里，在我们编写模块的逻辑中，这个 `send` 函数，本身没有必要说第一个参数必须是自身的不可变引用，按照逻辑来讲，它应该是一个可变引用，因为它调用了它自身内部的成员确实是应该跟着变化的。

所以说，这个例子就是为了展示内部可变性，强行搞出来的。因为按照逻辑，这个messenger应该是可变的。

但是我这里试着把全部的messenger相关的声明改成了可变性，是可以的，但是需要改动非常多的代码，我在想，这里会不会是这样子的，就是代价，该模块其实大部分是不可变的，为了在这里进行可变性操作强行使真个类变成个可变的需要的代价太大， 因此只在这里对不可变类的其中一个需要可变的成员进行内部可变性操作。

原文档中说了一句话：

> 然而，使用 `RefCell` 使得在只允许不可变值的上下文中编写修改自身以记录消息的 mock 对象成为可能。

反正我是没看出来为什么 _只允许不可变值的上下文_，我觉得可以可变。

stop，不说这里了，继续说怎么改，首先引入 `use std::cell::RefCell;` 声明 `msg_pool` 时，用该智能指针包裹。

此时，`msg_pool` 是智能指针类型，需要在其上使用 `borrow()` 和 `borrow_mut()` 获取它的不可变引用和可变引用。

修改后的代码，非测试模块代码没有变：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        msg_pool: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new () -> MockMessenger{
            MockMessenger {
                msg_pool: RefCell::new(vec![]),
            }
        }
    }

    impl MessengerTrait for MockMessenger {
        fn send(&self, msg: &str) {
            self.msg_pool.borrow_mut().push(String::from(msg));
        }
    }

    #[test]
    fn func () {
        let messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&messenger, 100);
        limit_tracker.set_value(20);
        assert_eq!(messenger.msg_pool.borrow()[0], "now value is 20");
    }
}
```

### 完整代码

```rust
pub trait MessengerTrait {
    fn send (&self, msg: &str);
}

pub struct LimitTracker<'a, T: MessengerTrait> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T> 
    where T: MessengerTrait
{
    pub fn new (messenger: &T, max: usize) -> LimitTracker<T>{
        LimitTracker {
            messenger,
            max,
            value: 0,
        }
    }

    pub fn set_value (&mut self, value: usize){
        self.value = value;
        self.messenger.send(&(format!("now value is {}", value))[..]);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        msg_pool: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new () -> MockMessenger{
            MockMessenger {
                msg_pool: RefCell::new(vec![]),
            }
        }
    }

    impl MessengerTrait for MockMessenger {
        fn send(&self, msg: &str) {
            self.msg_pool.borrow_mut().push(String::from(msg));
        }
    }

    #[test]
    fn func () {
        let messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&messenger, 100);
        limit_tracker.set_value(20);
        assert_eq!(messenger.msg_pool.borrow()[0], "now value is 20");
    }
}
```

## `RefCell<T>` 在运行时记录借用

当创建不可变和可变引用时，我们分别使用 `&` 和 `&mut` 语法。对于 `RefCell<T>` 来说，则是 `borrow` 和 `borrow_mut` 方法，这属于 `RefCell<T>` 安全 API 的一部分。`borrow` 方法返回 `Ref<T>` 类型的智能指针，`borrow_mut` 方法返回 `RefMut` 类型的智能指针。这两个类型都实现了 `Deref`，所以可以当作常规引用对待。

`RefCell<T>` 记录当前有多少个活动的 `Ref<T>` 和 `RefMut<T>` 智能指针。每次调用 `borrow`，`RefCell<T>` 将活动的不可变借用计数加一。当 `Ref<T>` 值离开作用域时，不可变借用计数减一。就像编译时借用规则一样，`RefCell<T>` 在任何时候只允许有多个不可变借用或一个可变借用。

如果在运行时发现，RefCell 违反了借用规则，则会抛出一个panic。

## 结合 `Rc<T>` 和 `RefCell<T>` 来拥有多个可变数据所有者

`RefCell<T>` 的一个常见用法是与 `Rc<T>` 结合。回忆一下 `Rc<T>` **允许对相同数据有多个所有者**，不过**只能提供数据的不可变访问**。如果有一个储存了 `RefCell<T>` 的 `Rc<T>` 的话，就可以得到有多个所有者 并且 可以修改的值了！

例如，回忆第15-1章的 cons list 的例子中使用 `Rc<T>` 使得多个列表共享另一个列表的所有权。因为 `Rc<T>` 只存放不可变值，所以一旦创建了这些列表值后就不能修改。让我们加入 `RefCell<T>` 来获得修改列表中值的能力。下例展示了通过在 Cons 定义中使用 `RefCell<T>`，我们就允许修改所有列表中的值了：

```rust
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(10)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

这里创建了一个 `Rc<RefCell<i32>>` 实例并储存在变量 value 中以便之后直接访问。接着在 a 中用包含 value 的 Cons 成员创建了一个 List。需要克隆 value 以便 a 和 value 都能拥有其内部值 5 的所有权，而不是将所有权从 value 移动到 a 或者让 a 借用 value。

我们将列表 a 封装进了 `Rc<T>` 这样当创建列表 b 和 c 时，他们都可以引用 a，正如示例 15-18 一样。

一旦创建了列表 a、b 和 c，我们将 value 的值加 10。为此对 value 调用了 borrow\_mut，这里使用了第五章讨论的自动解引用功能（“-&gt; 运算符到哪去了？” 部分）来解引用 `Rc<T>` 以获取其内部的 `RefCell<T>` 值。`borrow_mut` 方法返回 `RefMut<T>` 智能指针，可以对其使用解引用运算符并修改其内部值。

这是非常巧妙的！通过使用 `RefCell<T>`，我们可以拥有一个表面上不可变的 List，不过可以使用 `RefCell<T>` 中提供内部可变性的方法来在需要时修改数据。`RefCell<T>` 的运行时借用规则检查也确实保护我们免于出现数据竞争——有时为了数据结构的灵活性而付出一些性能是值得的。

标准库中也有其他提供内部可变性的类型，比如 `Cell<T>`，它类似 `RefCell<T>` 但有一点除外：它并非提供内部值的引用，而是把值拷贝进和拷贝出 `Cell<T>`。还有 `Mutex<T>`，其提供线程间安全的内部可变性，我们将在第 16 章中讨论其用法。请查看标准库来获取更多细节关于这些不同类型之间的区别。

## 附录：通过 `RefCell<T>` 在运行时检查借用规则

这一部分有点怪，放在前面讲，感觉很脱节，放在后面也不影响阅读。

~~不同于 `Rc<T>`，`RefCell<T>` 代表其数据的唯一的所有权 。那么是什么让 `RefCell<T>` 不同于像 `Box<T>` 这样的类型呢？~~

回忆一下第四章所学的借用规则：

* 在任意给定时刻，只能拥有一个可变引用或任意数量的不可变引用 之一（而不是两者）。
* 引用必须总是有效的。

相比于 引用 和 Box。`RefCell`作用于运行时。前者如果违反借用规则，则在编译阶段报错，后者则会在运行阶段抛出一个panic。

看一看借用规则在不同阶段运用的区别：

| 编译时 | 运行时 |
| :---: | :---: |
| 尽早暴露问题 | 问题可能在生产环境中发生 |
| 没有运行时开销 | 因借用计数产生性能损失 |
| 是Rust默认行为 | 实现特定的内存安全场景\(不可变环境中修改自身数据\) |

可以看出，引用 和 Box智能指针的优点要远多于 RCT，但是，**运行时检查借用规则的好处则是允许出现特定内存安全的场景，而它们在编译时检查中是不允许的**。静态分析，正如 Rust 编译器，是天生保守的。但代码的一些属性不可能通过分析代码发现：其中最著名的就是 停机问题（Halting Problem），这超出了本书的范畴。

{% hint style="info" %}
要注意的是，`Rc<T>`和`RefCell<T>` 只能用于单线程场景。
{% endhint %}

### 那么如何选择智能指针？

|  | `Box<T>` | `Rc<T>` | `RefCell<T>` |
| :---: | :---: | :---: | :---: |
| 数据的所有者数量 | 一个 | 多个 | 一个 |
| 可以存在哪些可变性借用 | 可变、不可变借用 | 不可变借用 | 可变、不可变借用 |
| 借用检查发生时机 | 编译时 | 编译时 | 运行时 |



