# 10-1 泛型数据类型

我们可以使用泛型为像函数签名或结构体这样的项创建定义，这样就可以用于不同的具体数据类型。让我们看看如何使用泛型定义函数、结构体、枚举和方法，然后我们将讨论泛型如何影响代码性能。

## 函数定义中使用泛型

一个栗子：寻找 slice 中的最大值

```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn largest_char(list: &[char]) -> char {
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

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);
    assert_eq!(result, 100);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
    assert_eq!(result, 'y');
}
```

可以看到这两个函数除了名称、数据类型不同以外，其他并无差别，因此**这里使用泛型进行改写**。在函数签名中使用泛型的时候必须先使用尖括号 `<>`声明它，如果不这样做，编译器会不知道这里的 `T` 是泛型还是其他东西：

```rust
fn largest<T> (list: &[T]) -> T
```

这有点类似于课本中的伪定义了，这里的 `T` 是指，此处可以接受任何类型，并且所有的 `T` 都是这个类型。新的函数定义统一了之前的两个函数，只不过暂时该代码还不能通过编译，因为 `>` 不一定可以应用在类型 `T` 上。

编译器会提示：

```text
note: an implementation of `std::cmp::PartialOrd` might be missing for `T`
```

> 注释中提到了 `std::cmp::PartialOrd`，这是一个 trait。下一部分会讲到 trait。不过简单来说，这个错误表明 largest 的函数体不能适用于 T 的所有可能的类型。因为在函数体需要比较 T 类型的值，不过它只能用于我们知道如何排序的类型。为了开启比较功能，标准库中定义的 `std::cmp::PartialOrd` trait 可以实现类型的比较功能（查看附录 C 获取该 trait 的更多信息）。
>
> 标准库中定义的 `std::cmp::PartialOrd` trait 可以实现类型的比较功能。在 “trait 作为参数” 部分会讲解如何指定泛型实现特定的 trait，不过让我们先探索其他使用泛型参数的方法。

## 结构体定义中的泛型

一个例子：

```cpp
struct Point<T> {
    x: T,
    y: T,
}

let integer = Point {x:5, y:6};
let f = Point {x: 1.2, y: 2.3};
let error = Point {x: 3, y: 1.2} // error
```

注意这里的结构体定义中只使用了一个泛型，且 x, y 必须是同类型，**如果想使用不同的类型，可以定义多个泛型**。

```cpp
struct Point<T, U> {
    x: T,
    y: U,
}
let integer_and_float = Point { x: 5, y: 4.0 };
```

## 枚举定义中的泛型

对于枚举泛型，我们曾在第六章中学过 `Option<T>` ：

```rust
enum Option<T> {
    Some<T>,
    None,
}
```

现在就更好理解了。`Option` 是一个拥有泛型 `T` 的枚举，它有两个成员：

* 成员 `Some` 它存放了一个类型为 `T` 的值
* 成员 `None` 它不存放任何值

枚举也可以有多个泛型类型。例如第九章使用过的 Result 枚举：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Result 枚举有两个泛型类型，`T` 和 `E`。具体来说有两个成员：

* `Ok`，存放一个类型 `T` 的值
* `Err`，存放一个类型 `E` 的值

这个定义使得 Result 枚举能很方便的表达任何可能成功（返回 `T` 类型的值）也可能失败（返回 `E` 类型的值）的操作。实际上，这就是我们在示例 9-3 用来打开文件的方式：当成功打开文件的时候，T 对应的是 `std::fs::File` 类型；而当打开文件出现问题时，`E` 的值则是 `std::io::Error` 类型。

当你意识到代码中定义了多个结构体或枚举，它们不一样的地方只是其中的值的类型的时候，不妨通过泛型类型来避免重复。

## 方法定义中的类型

在Rust中有意地区分了函数 和 方法，其实两者并没有什么区别，我看到很多新手在网上像文字狱一样纠正别人的概念，属实是没有必要。

在为结构体和枚举实现方法时，（回顾第五章）一样可以使用泛型：

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x (&self) -> &T {
        &self.x
    }
}

fn main () {
    let p = Point {x:5, y:10};
    println!("p.x = {}", p.x());
}
```

示例 10-9：在 `Point<T>` 结构体上实现方法 `x`，它返回 `T` 类型的字段 `x` 的引用。

{% hint style="info" %}
其实我这里不是很明白为什么返回 T 类型字段的引用。比如在此案例里，由于在创建实例p时，x传入了一个整型，那么泛型T会默认设为 i32 类型，那么返回值的类型不应该是 i32 类型吗？而且 i32 不是一个标量类型吗？为什么能返回 &i32。
{% endhint %}

注意，必须在 `impl` 后面声明 `T`，这样就可以在 `Point<T>` 上实现的方法中使用它了。在 `impl` 之后声明泛型 `T`，这样Rust就知道 `Point` 的尖括号类型是泛型而不是具体类型。

例如，可以选择为 Point 实例实现方法，只有当 Point 泛型类型为 f32 时，才拥有此方法。

```rust
impl Point<f32> {
    fn distance_from_origin (&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

并且，结构体定义中的泛型，和结构体方法中的泛型，可以不使用同一种泛型。下面的例子就是把两种不同类型的泛型组合起来。

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> { // 这一行是要与上面的结构体定义一致
    // 而这一行可以声明新的泛型
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c'};
    let p3 = p1.mixup(p2);
    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

## 泛型代码的性能

好消息是Rust并不会因为泛型这种复杂的多类型而拖累速度，因为Rust在编译时会对泛型代码进行 **单态化**。

简单说就是，Rust 会在真正向填入泛型类型时，进行单态化。编译器生成的单态化版本的代码看起来像这样，并包含将泛型 `Option<T>` 替换为编译器创建的具体定义后的用例代码

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

我们可以使用泛型来编写不重复的代码，而 Rust 将会为每一个实例编译其特定类型的代码。这意味着在使用泛型时没有运行时开销；当代码运行，它的执行效率就跟好像手写每个具体定义的重复代码一样。这个单态化过程正是 Rust 泛型在运行时极其高效的原因。

