# 4-1 所有权 ownership

所有权我理解为是和内存管理相关的概念。

对于部分语言来说，垃圾回收机制是自动的，不需要我们程序员去关心，比如 java 等。对于有的语言来说，垃圾回收是需要程序员自主管理的。

而RUST是第三种方式：

> 通过所有权系统管理内存，编译器在编译时会根据一系列的规则进行检查。在运行时，所有权系统的任何功能都不会减慢程序。

## 栈 和 堆

我觉得要理解所有权必须要对 stack 和 heap 有一个清晰的认识。

栈空间，想象为连续的固定的空间，像中药柜一样，先进后出。一般标量类型会存储在栈空间。

堆空间，缺乏组织，在往堆存放数据时，要请求一定大小的空间，OS找到适合大小的一片空间，把它标记为已使用，并返回指向该空间的指针，这个过程称作**在堆上分配内存**，简称为 分配 \(allocation\)。将数据放入栈空间不认为是分配，因为指针的大小是已知并且固定的。一般复合类型数据会存储在堆空间，比如数组。

入栈比在堆上分配内存要快，因为（入栈时）操作系统无需为存储新数据去搜索内存空间；其位置总是在栈顶。相比之下，在堆上分配内存则需要更多的工作，这是因为操作系统必须首先找到一块足够存放数据的内存空间，并接着做一些记录为下一次分配做准备。

访问堆上的数据比访问栈上的数据慢，因为必须通过指针来访问。现代处理器在内存中跳转越少就越快（缓存）。

## 所有权规则

牢记：

1. RUST 中每一个值都有一个称为 **所有者 owner** 的变量。
2. 一个值在任一时刻有且只有一个所有者。
3. 当所有者离开作用域时，这个对应的值被丢弃。

例子：

```javascript
{    // 作用域开始
    // s无效
    let s = "hello"; // s有效
    // "hello" 是值
    // s 是 "hello"的所有者
}    // 作用域开始结束，s 以及hello被丢弃
```

## String 类型

为了探究所有权，我们必须引入一个复杂的数据类型，相比于固定长度大小的字符串字面量值，String 类型更加复杂。

String 被存储在堆上，能够存储未知大小的文本，可以使用 from 从字符串字面量值创建String。

```javascript
let mut s = String::from("hello");

s.push_str(", world!"); // push_str() 在字符串后追加字面值

println!("{}", s); // 将打印 `hello, world!`
```

## 内存与分配

> 就字符串字面值来说，我们在编译时就知道其内容，所以文本被直接硬编码进最终的可执行文件中。这使得字符串字面值快速且高效。不过这些特性都只得益于字符串字面值的不可变性。不幸的是，我们不能为了每一个在编译时大小未知的文本而将一块内存放入二进制文件中，并且它的大小还可能随着程序运行而改变。

对于 String 类型，为了支持一个可变，可增长的文本片段，需要在堆上分配一块在编译时未知大小的内存来存放内容。这意味着：

* 必须在运行时向操作系统请求内存。
* 需要一个当我们处理完 String 时将内存返回给操作系统的方法。

第一部分由我们完成：当调用 `String::from` 时，它的实现 \(implementation\) 请求其所需的内存。这在编程语言中是非常通用的。

> 然而，第二部分实现起来就各有区别了。在有 垃圾回收（garbage collector，GC）的语言中， GC 记录并清除不再使用的内存，而我们并不需要关心它。没有 GC 的话，识别出不再使用的内存并调用代码显式释放就是我们的责任了，跟请求内存的时候一样。从历史的角度上说正确处理内存回收曾经是一个困难的编程问题。如果忘记回收了会浪费内存。如果过早回收了，将会出现无效变量。如果重复回收，这也是个 bug。我们需要精确的为一个 allocate 配对一个 free。

Rust 采取了一个不同的策略：内存在拥有它的变量离开作用域后就被自动释放。

```javascript
{
    let s = String::from("hello"); // 从此处起，s 是有效的
    // 使用 s
}                                  // 此作用域已结束，
                                   // s 不再有效
```

这是一个将 String 需要的内存返回给操作系统的很自然的位置：当 s 离开作用域的时候。当变量离开作用域，Rust 为我们调用一个特殊的函数。这个函数叫做 `drop`，在这里 String 的作者可以放置释放内存的代码。Rust 在结尾的 `}` 处自动调用 drop。

> 注意：在 C++ 中，这种 item 在生命周期结束时释放资源的模式有时被称作 资源获取即初始化（Resource Acquisition Is Initialization \(RAII\)）。如果你使用过 RAII 模式的话应该对 Rust 的 drop 函数并不陌生。

## \*变量与数据的交互方式（一）：移动

对于标量：

```javascript
let x = 5;
let y = x;
```

这种方式先把5存入栈中，并返回指针到x，然后又复制了一个新的值给到了y。这时，内存中就有两个5，这就相当于Js中的基本数据类型。

对于复合变量：

```javascript
let s1 = String::from("hello");
let s2 = s1;
```

对于字符串的值，会以 index - value 的形式存储在堆上，而所谓的String 类型的所有者，并不是直接指向这个值，而是指向表示String的一张表。其中包含 `name、ptr、len、cap` 等字段。`s1` 存储在栈上，而实际内容存储在堆上。

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/2ced80009dcc4ff8a2c1884dd5d3109e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_10,color_FFFFFF,t_70,g_se,x_16)

当我们将 s1 赋值给 s2，String 的数据被复制了，这意味着我们从栈上拷贝了它的指针、长度和容量。我们并没有复制指针指向的堆上数据。如图所示：

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/e432cdc6240b45b1acb0fa7909595e63.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_10,color_FFFFFF,t_70,g_se,x_16) 之前我们提到过当变量离开作用域后，Rust 自动调用 drop 函数并清理变量的堆内存。不过上图展示了两个数据指针指向了同一位置。**这就有了一个问题：当 s2 和 s1 离开作用域，他们都会尝试释放相同的内存。这是一个叫做 二次释放（double free）的错误，也是之前提到过的内存安全性 bug 之一。**两次释放（相同）内存会导致内存污染，它可能会导致潜在的安全漏洞。

因此，对于Rust，与其尝试复制被分配的内存，**Rust会直接认为 `s1` 失效了！** 因此，下面这段代码不能运行。

```javascript
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

你会得到一个类似如下的错误，因为 Rust 禁止你使用无效的引用。

```bash
error[E0382]: use of moved value: `s1`
 --> src/main.rs:5:28
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |
5 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`, which does
  not implement the `Copy` trait
```

**如果你在其他语言中听说过术语 浅拷贝（shallow copy）和 深拷贝（deep copy），那么拷贝指针、长度和容量而不拷贝数据可能听起来像浅拷贝。不过因为 Rust 同时使第一个变量无效了，这个操作被称为 移动（move），而不是浅拷贝。上面的例子可以解读为 s1 被 移动 到了 s2 中。**

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/d559b34dca1b43839654e9c32bbc3d21.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_10,color_FFFFFF,t_70,g_se,x_16) 这样就解决了我们的问题！因为只有 s2 是有效的，当其离开作用域，它就释放自己的内存，完毕。

另外，这里还隐含了一个设计选择：Rust 永远也不会自动创建数据的 “深拷贝”。因此，任何 自动 的复制可以被认为对运行时性能影响较小。

## 变量与数据交互的方式（二）：克隆

如果我们确实需要实现对复杂数据的深拷贝，可以使用一个叫做 `clone` 的通用函数。该方法不仅会对栈上的数据、**也会对堆上的数据进行拷贝**。

```javascript
fn main() {
let s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 = {}, s2 = {}", s1, s2);
}
```

![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/efd890a3b99d4d12a7e12661b62984aa.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_9,color_FFFFFF,t_70,g_se,x_16) 要注意的是，`clone` 的执行有可能相当消耗资源，谨慎使用。

## 只在栈上的数据拷贝

先看个例子：

```javascript
let x = 5;
let y = x;
```

此时 `x、y` 都是有效的，看上去似乎与我们上面的内容矛盾：没有调用 `clone`，不过 `x` 依然有效且没有被移动到 `y` 中。

对于标量数据结构进行拷贝的时候，是迅速的，因为编译时就已知大小，没有深浅拷贝的说法。

**Rust有一个叫做 `Copy Trait` 的特殊注解，可以用在类似整形这样的存储在栈的类型上**。

* 如果一个类型拥有 `Copy Trait` ，一个旧变量赋值给新变量后，依然可用。
* 不允许自身或其部分实现了 `Drop Trait` 的类型使用 Copy Trait。

**那么什么类型是可以拥有 `Copy Trait` 的呢？** 可以通过文档查阅，不过，有一个通用规则：**任何简单的标量的组合是可以 `Copy` 的，不需要分配内存或某种形式资源的类型是 `Copy` 的。**

* 所有整数类型，比如 u32。
* 布尔类型，bool，它的值是 true 和 false。
* 所有浮点数类型，比如 f64。
* 字符类型，char。
* 元组，当且仅当其包含的类型也都是 Copy 的时候。比如，\(i32, i32\) 是 Copy 的，但 \(i32, String\) 就不是。

## 所有权与函数

在RUST中，将值传递给函数在语义上与赋值（移动）相似。向函数传递值会造成 **移动** 或 **克隆**。

* 复杂数据结构在传参时，会发生 `move` ，旧变量会失效
* 简单数据结构在传参时，也会发生 `move`，旧变量不会失效

```javascript
fn main() {
    let s = String::from("hello");  // s 进入作用域
    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以s到这里不再有效
    let x = 5;                      // x 进入作用域
    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，所以在后面可继续使用 x
} // 这里, x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 所以不会有特殊操作

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。不会有特殊操作
```

## 返回值与作用域

返回值也可以转移所有权。从函数内部转移给接收函数返回值的变量，此时数据不会被drop。

但是，在每一个函数中都获取所有权并接着返回所有权有些啰嗦。如果我们还要接着使用它的话，每次都传进去再返回来就有点烦人了。如果我们想要函数使用一个值但不获取所有权该怎么办呢？

```javascript
fn main() {
    let s1 = String::from("hello");
    let (s2, len) = calculate_length(s1);
    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() 返回字符串的长度
    (s, length)
}
```

上述代码显得十分冗余，但是这种需求很常见。幸运的是，Rust提供了一个功能叫做 **引用 references**。

