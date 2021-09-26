# 第十章 泛型、trait 与生命周期

每一个编程语言都有高效处理重复概念的工具，Rust 中的工具之一就是**泛型\(generics\)。**

同理为了编写一份可以用于多种具体值的代码，函数并不知道其参数为何值，这时就可以让函数获取泛型而不是像 `i32` 或 `String` 这样的具体类型。我们已经使用过第六章的 `Option<T>`，第八章的 `Vec<T>` 和 `HashMap<K, V>`，以及第九章的 `Result<T, E>` 这些泛型了。本章会探索如何使用泛型定义我们自己的类型、函数和方法！

之后，我们讨论 **trait**，这是一个定义泛型行为的方法。trait 可以与泛型结合来将泛型限制为拥有特定行为的类型，而不是任意类型。

最后介绍 **生命周期**（_lifetimes_），它是一类允许我们向编译器提供引用如何相互关联的泛型。Rust 的生命周期功能允许在很多场景下借用值的同时仍然使编译器能够检查这些引用的有效性。

## **提取函数来减少重复**

在介绍泛型之前，先回顾一个古老的技术来减少重复——提取函数。

```rust
fn main () {
    let number_list = vec![34,50,23,100,54];
    let mut largest = number_list[0];
    
    for number in number_list {
        if number > number_list {
            largest = number;
        }
    }
    println!("The largest number is {}", largest);
}
```

上述的例子是寻找数组中最大的数，但是这个代码不能复用，让我们提取一个函数。

```rust
fn main () {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let number_list = vec![102, 34, 6000, 89, 54, 2, 43, 8];
    let result = largest(&number_list);
    println!("The largest number is {}", result);
}

fn largest (list: &[i32]) -> i32 {
    let mut largest = list[0];
    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

在不同的场景使用不同的方式，我们也可以利用相同的步骤和泛型来减少重复代码。与函数体可以在抽象`list`而不是特定值上操作的方式相同，泛型允许代码对抽象类型进行操作。

如果我们有两个函数，一个寻找一个 `i32` 值的 slice 中的最大项而另一个寻找 `char` 值的 slice 中的最大项该怎么办？该如何消除重复呢？让我们拭目以待！

