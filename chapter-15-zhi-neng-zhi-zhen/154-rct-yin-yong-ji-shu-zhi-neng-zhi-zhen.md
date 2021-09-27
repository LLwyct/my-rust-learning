# 15-4 Rc&lt;T&gt;引用计数智能指针

## `Rc<T>`的基本概念

`Rc<T>` 和 所有权有一点关系。

大部分情况下所有权关系是非常明确的：可以知道哪个变量拥有哪个值。回顾一下所有权规则：

1. RUST 中每一个值都有一个称为 所有者 owner 的变量。
2. 一个值在任一时刻有且只有一个所有者。
3. 当所有者离开作用域时，这个对应的值被丢弃。

但是有时，单个值会有多个_所有者_ 。当一个值的所有者数量没有归零时，这个值不应该被清除。

为了**启用多所有权**，Rust 有一个 `Rc<T>` 类型，称为 **引用计数**\(reference counting\)。

## `Rc<T>`的使用场景

`Rc<T>` 用于当我们希望在堆上分配一些内存供程序的多个部分读取，而且无法在编译时确定程序的哪一部分会最后结束使用它的时候。如果确实知道哪部分是最后一个结束使用的话，就可以令其成为数据的所有者，正常的所有权规则就可以在编译时生效。

注意 `Rc<T>` **只能用于单线程场景**；第十六章并发会涉及到如何在多线程程序中进行引用计数。

## 使用`Rc<T>`共享数据

 

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/53cbddcebac14a718e47a14043fe1c2e.png)

我们之前在 Box 智能指针章节中学习了——如何使用`Box`指针和枚举类型实现一个Cons List。

那时创建的链表类似于上图中的 a 链表，今天在这一节使用 `Rc` 智能指针实现一个上面这种链表。

### 为什么 Box 智能指针不行？

因为所有权，当我们创建 a 之后，把 a 接到 b 的后面时，相当于把a的所有权移交给了 b，发生了move，此时 a 已经不再拥有数据了，因此，把 a 再一次接到 c 的后面时就会报错 `error[E0382]: use of moved value: a`。

### Rc 智能指针的使用方法

在实现该例前，介绍一下**如何创建 Rc 智能指针**，由于 `Rc` 不是preclude ：

```rust
use std::rc::Rc;
let p = Rc::new(data);
```

data 就是需要多重所有者的数据，p就是data的引用计数指针（地址）。

### 实现示例

#### 第一步 创建链表 a

此时就创建了 a 的实例链表。什么意思呢？你可以理解为，a 此时直接指向的是 该链表实际在堆上的数据。

```rust
use std::rc::Rc;
use crate::List::{Cons, Nil};

enum List {
    Cons(i32, Rc<List>),
    Nil,
}

let a: List = Cons(5, Rc::new(Cons(10, Rc::new(Nil))));
```

#### 第二步 创建链表 b 和 c

```rust
let b: List = Cons(3, ?);
let b: List = Cons(4, ?);
```

#### 第三步 把 a 插入b、c

如果把 a 直接附在 b 的后面，a 就不能用了。

```rust
let a = Cons(5, Rc::new(Cons(10, Rc::new(Nil))));
// 这里多写一步是为了强化需要使用Rc::new创建指针的概念
let smart_pointer_of_a = Rc::new(a);
let b = Cons(3, smart_pointer_of_a);
let c = Cons(4, smart_pointer_of_a);
```

其实直到这里，用 C语言的思路完全说的通，`smart_pointer_of_a` 是 a 链表真实数据的指针，现在把这个指针插入b、c之后有什么问题？明明没有问题！

**但是在Rust中，就像我们在4-1所有权一节所学到的：指针（地址）在进行赋值操作之后，不会使两个引用指向同一个内存区域，而是后者会夺取前者的所有权，发生了move操作**。

> 我的话：类比 Js，Js引用类型在赋值之后，两个变量都会指向同一个真实的数据
>
> **这里我还要强调**：虽然Rc智能指针说是可以实现多重所有权，但是，不是只要使用Rc智能指针就直接可以多重所有权了。要不然代码到这里已经可以编译通过了。

在本例中`smart_pointer_of_a`其实本质上也是一个指针，在这里赋值给了 b 链表元组枚举成员的 第二项之后，相当于做了move操作。

因此，在这里不能让它简简单单就这么move了，要怎么做呢？要利用 `Rc` 智能指针特殊提供的魔法！—— `Rc::clone()`。

```rust
let b = Cons(3, Rc::clone(&smart_pointer_of_a));
let c = Cons(4, Rc::clone(&smart_pointer_of_a));
```

强调一下这里使用 `Rc::clone()` 的必要性

* 不会任由它发生move，使 a 丢失所有权
* `Rc::clone` 只会增加引用计数，进行了简单的浅拷贝，虽然内部实现和传统的浅拷贝必有区别，但是性能很好，不深究

再思考，这里其实还是使用了拷贝（没有直接赋值），才让所有权没有发生move，只不过`Rc`智能指针提供的拷贝提供了额外的引用计数功能。那么是不是还有别的方法？

没错，也可以直接调用变量的`clone`，只不过会发生深拷贝，在这里完全没有必要这么做。

```rust
let b = Cons(3, smart_pointer_of_a.clone());
let c = Cons(4, smart_pointer_of_a.clone());
```

#### 完整代码

```rust
use std::rc::Rc;
use crate::List::{Cons, Nil};

enum List {
    Cons(i32, Rc<List>),
    Nil,
}

fn main () {
    let a = Cons(5, Rc::new(Cons(10, Rc::new(Nil))));
    // 这里多写一步是为了强化需要使用Rc::new创建指针的概念
    let smart_pointer_of_a = Rc::new(a);
    let b = Cons(3, Rc::clone(&smart_pointer_of_a));
    let c = Cons(4, Rc::clone(&smart_pointer_of_a));
}
```

## 克隆 `Rc<T>` 会增加引用计数

`Rc::strong_count(&T)` 方法可以获得指针\(引用\)的**强引用计数**。

```rust
let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
println!("count after creating a = {}", Rc::strong_count(&a));
```

## 小节总结

上面说了这么多其实就还是不可变引用，`Rc<T>` 允许在程序的多个部分之间只读地共享数据。如果 `Rc<T>` 也允许多个可变引用，则会违反第四章讨论的借用规则之一：相同位置的多个可变借用可能造成数据竞争和不一致。不过可以修改数据是非常有用的！在下一部分，我们将讨论内部可变性模式和 `RefCell<T>` 类型，它可以与 `Rc<T>` 结合使用来处理不可变性的限制。

