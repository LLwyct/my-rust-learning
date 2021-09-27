# 4-2 引用 与 借用

## 引用 与 借用 references and borrowing

上篇遗留代码存在这样一个问题：为了不丢失获得String的句柄，我们不得不先把字符串所有权转移到函数内，在函数最后再转移回主函数，因此我们不得不返回一个元组。

```javascript
fn main() {
    let s1 = String::from("hello");
    let (s2, len) = calculate_length(s1);
}
fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() 返回字符串的长度
    (s, length)
}
```

下面定义了一个新、具有的同样作用的函数：

```javascript
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

& 符号就是引用，**它们允许你使用值但不获取其所有权**，如图所示：

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/098dd2d4eacd41caa2becb41c408c42a.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_20,color_FFFFFF,t_70,g_se,x_16) 仔细看看这个函数调用：

```javascript
let s1 = String::from("hello");
let len = calculate_length(&s1);
```

`&s1` 语法让我们创建一个 指向 值 `s1` 的引用，但是并不拥有它。因为并不拥有这个值，当引用离开作用域时其指向的值也不会被丢弃。

同理，函数签名使用 `&` 来表明参数 `s` 的类型是一个引用。让我们增加一些解释性的注释：

```javascript
fn calculate_length(s: &String) -> usize { // s 是对 String 的引用
    s.len()
} // 这里，s 离开了作用域。但因为它并不拥有引用值的所有权，
  // 所以什么也不会发生
```

变量 `s` 有效的作用域与函数参数的作用域一样，**不过当引用离开作用域后并不丢弃它指向的数据，因为我们没有所有权**。**当函数使用引用而不是实际值作为参数，无需返回值来交还所有权，因为就不曾拥有所有权**。

我们将获取引用作为函数参数称为 **借用（borrowing）**。正如现实生活中，如果一个人拥有某样东西，你可以从他那里借来。当你使用完毕，必须还回去。

要注意的是，借用的变量默认是不可修改的。

```javascript
fn main() {
    let s = String::from("hello");
    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world");
}
error[E0596]: cannot borrow immutable borrowed content `*some_string` as mutable
 --> error.rs:8:5
  |
7 | fn change(some_string: &String) {
  |                        ------- use `&mut String` here to make mutable
8 |     some_string.push_str(", world");
  |     ^^^^^^^^^^^ cannot borrow as mutable
```

## 可变引用

对上述代码进行一些小改动即可可变：

```javascript
fn main() {
    let mut s = String::from("hello");
    change(&mut s);
}
fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

首先 `s` 必须是 `mut` 的，然后创建一个可变引用，和一个接收可变引用的函数。

**不过可变引用有一个很大的限制：在特定作用域中的特定数据只能有一个可变引用**。这些代码会失败：

```javascript
let mut s = String::from("hello");
let r1 = &mut s;
let r2 = &mut s;
```

这个限制的好处是 Rust 可以在编译时就避免数据竞争。数据竞争（data race）类似于竞态条件，它可由这三个行为造成：

* 两个或更多指针同时访问同一数据。
* 至少有一个指针被用来写入数据。
* 没有同步数据访问的机制。

数据竞争会导致未定义行为，难以在运行时追踪，并且难以诊断和修复；Rust 避免了这种情况的发生，因为它甚至不会编译存在数据竞争的代码！

如既往，**可以使用大括号来创建一个新的作用域，以允许拥有多个可变引用**，只是不能 同时 拥有：

```javascript
let mut s = String::from("hello");
{
    let r1 = &mut s;
} // r1 在这里离开了作用域，所以我们完全可以创建一个新的引用
let r2 = &mut s;
```

**也不能同时拥有一个数据的可变引用和不可变引用：**

```javascript
let mut s = String::from("hello");
let r1 = &s; // 没问题
let r2 = &s; // 没问题，不可变引用可以有多个
let r3 = &mut s; // 大问题
```

一个数据的引用的作用域会在最后一次使用后停止。有的人这里可能就晕了，怎么知道是不是最后一次使用呢？

因为，引用的声明和使用，由于其作用域下的所有权，是静态，在编译的时候就可以敲定哪里是最后一次使用。**因此，只要不可变引用的最后一次使用在可变引用之前，代码就可以编译。**

## 悬垂引用（Dangling References）

> 在具有指针的语言中，很容易通过释放内存时保留指向它的指针而错误地生成一个 **悬垂指针（dangling pointer）**，所谓悬垂指针是其指向的内存可能已经被分配给其它持有者。相比之下，在 Rust 中编译器确保引用永远也不会变成悬垂状态：当你拥有一些数据的引用，编译器确保数据不会在其引用之前离开作用域。

让我们尝试创建一个悬垂引用，Rust 会通过一个编译时错误来避免：

```javascript
fn main() {
    let reference_to_nothing = dangle();
}
fn dangle() -> &String {
    let s = String::from("hello");
    &s
}
```

> 错误信息引用了一个我们还未介绍的功能：生命周期（lifetimes）。第十章会详细介绍生命周期。不过，如果你不理会生命周期部分，错误信息中确实包含了为什么这段代码有问题的关键信息。

**因为 s 是在 dangle 函数内创建的，当 dangle 的代码执行完毕后，s 将被释放。不过我们尝试返回它的引用。这意味着这个引用会指向一个无效的 String，这可不对！Rust 不会允许我们这么做。**

这里的解决方法是直接返回 `String`：

```javascript
fn no_dangle() -> String {
    let s = String::from("hello");
    s
}
```

**这样就没有任何错误了。所有权被移动出去，所以没有值被释放。**

## 引用的规则

让我们概括一下之前对引用的讨论：

* 在特定作用域内，要么 只能有一个可变引用，要么 只能有多个不可变引用。
* 引用必须总是有效的。

接下来，我们来看看另一种不同类型的引用：slice。

