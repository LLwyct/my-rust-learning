# 15-2 Deref Trait

## Deref Trait

* 实现 `Deref` Trait 使我们可以自定义解引用运算符 `*` 的行为
* 通过实现 `Deref`，智能指针可像常规引用一样来处理

## 解引用运算符

{% hint style="info" %}
**强调一下：常规的引用也是一种指针！**
{% endhint %}

引用和智能指针的解引用运算符都是 `*`

### 引用下的解引用

```typescript
let x = 5;
let y = &x;
let z = *y; // 5
```

### Box 智能指针下的解引用

```typescript
let x = 5;
let y = Box::new(x);
let z = *y; // 5
```

## 定义自己的智能指针

`Box <T>` 本质上被定义成**只拥有一个元素的 tuple struct（元组结构体）**。

没什么好说的，定义一个元组结构体并实现一个 `Deref` Trait 即可。

```rust
use std::ops::Deref;

struct Mybox<T>(T);

impl<T> Mybox<T> {
    fn new (x: T) -> Mybox<T> {
        Mybox(x)
    }
}

impl <T> Deref for Mybox<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.0
    }
}

fn main () {
    let y = Mybox::new(5);
    assert_eq!(5, *y);
}
```

## 函数和方法的 隐式解引用转化 \(Deref Coercion\)

**隐式解引用转化** 是为函数和方法提供的一种便捷特性。

假设 `T` 实现了 `Deref` Trait，隐式解引用转化 可以把 `T` 的引用转化为 `T` 经过 Deref 操作后生成的引用。

这个真的牛逼，我看了好久才看懂。举个例子

```rust
fn hello (name: &str) {}

let m = Mybox::new(String::from("Rust"));
hello(&m);
```

看一下这个例子，传入 `hello` 函数的是 `&m`

`&m` 其实就是 `&Box`

再看原话："T的引用" -&gt; "\(经过 Deref 操作后生成的\)的引用"

那么在上例中

“T 的引用”是 `&Box`

那么 `T` 就是 `Box(String)`

"经过 Deref 操作后生成的"，`Box(String)` 经过Deref之后生成什么呢？

`Box(String)` 是智能指针，解引用之后是 `String` 类型。

“的引用”，`String` 的引用是什么，是 `&String`。

标准库中提供了 `String` 上的 Deref 实现，其会返回字符串 `slice`，这可以在 Deref 的 API 文档中看到。Rust 再次调用 **隐式解引用转化** 将 `&String` 变为 `&str`，这就符合 `hello` 函数的定义了。

如果 Rust 没有实现 **隐式解引用转化** ，为了使用 `&Box<String>` 类型的值调用 hello，则不得不编写如下代码：

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
}
```

## Deref 强制转换如何与可变性交互

之前讲解的都是使用 `Deref` trait 重载**不可变引用**的 `*` 运算符。

Rust 也提供了 `DerefMut` trait 用于重载**可变引用**的 `*` 运算符。

Rust 在发现类型和 trait 实现满足三种情况时会进行 `Deref` 强制转换：

* 当 `T: Deref<Target=U>` 时从 `&T` 到 `&U`。
* 当 `T: DerefMut<Target=U>` 时从 `&mut T` 到 `&mut U`。
* 当 `T: Deref<Target=U>` 时从 `&mut T` 到 `&U`。

头两个情况除了可变性之外是相同的：第一种情况表明如果有一个 `&T`，而 `T` 实现了返回 `U` 类型的 `Deref`，则可以直接得到 `&U`。第二种情况表明对于可变引用也有着相同的行为。

第三个情况有些微妙：Rust 也会将可变引用强转为不可变引用。但是反之是 不可能 的：不可变引用永远也不能强转为可变引用。因为根据借用规则，如果有一个可变引用，其必须是这些数据的唯一引用（否则程序将无法编译）。将一个可变引用转换为不可变引用永远也不会打破借用规则。将不可变引用转换为可变引用则需要数据只能有一个不可变引用，而借用规则无法保证这一点。因此，Rust 无法假设将不可变引用转换为可变引用是可能的。

