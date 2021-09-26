# 13-2 迭代器

## 创建一个迭代器

```rust
let v1 = vec![1,2,3];
let mut v1_iter = v.iter();
assert_eq!(v1_iter.next(), Some(&1));
assert_eq!(v1_iter.next(), Some(&2));
assert_eq!(v1_iter.next(), Some(&3));
assert_eq!(v1_iter.next(), None);
```

## Iterator trait 和 next方法

和 Js 一样，如果想为一个数据结构实现迭代器，需要实现一些固定的方法。比如 `Iterator` trait 和 `next` 方法。

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // 此处省略了方法的默认实现
}
```

和 Js 一样 `next` 是 `Iterator` 实现者被要求定义的唯一方法。`next` 一次返回迭代器中的一个项，封装在 `Some` 中，当迭代器结束时，它返回 `None`。

注意 `v1_iter` 需要是可变的：在迭代器上调用 `next` 方法改变了迭代器中用来记录序列位置的状态。换句话说，代码 **消费（consume）**了，或使用了迭代器。每一个 `next` 调用都会从迭代器中消费一个项。使用 `for`循环时无需使 `v1_iter`可变因为 `for` 循环会获取 `v1_iter` 的所有权并在后台使 `v1_iter` 可变。

另外需要注意到从 `next` 调用中得到的值是 vector 的不可变引用。`iter` 方法生成一个不可变引用的迭代器。如果我们需要一个获取 `v1` 所有权并返回拥有所有权的迭代器，则可以调用 `into_iter` 而不是 `iter`。类似的，如果我们希望迭代可变引用，则可以调用 `iter_mut` 而不是 `iter`。

## 消费迭代器的方法

之前说过迭代器是惰性的，必须要有消费迭代器的方法，才会调用 `next`， 调用`next`的方法被称为 **消费适配器 consuming adaptors**。

`Iterator trait` 中定义了另一类方法，被称为 **迭代器适配器（iterator adaptors）**，他们允许我们将当前迭代器变为不同类型的迭代器。可以链式调用多个迭代器适配器。**不过因为所有的迭代器都是惰性的，必须调用一个消费适配器方法以便获取迭代器适配器调用的结果**。

也就是说迭代消费器不能单独使用。

常见的消费适配器：

* `sum`、`collect`

常见的迭代器适配器：

* `map`

## 创建自定义迭代器

```rust
struct Counter {
    count: u32,
}

impl Counter {
    // new的实现不能在 Iterator 中，因为 Iter中并没有定义该方法
    fn new () -> Counter {
        Counter {
            count: 0,
        }
    }
}

impl Iterator for Counter {
    // 这里必须这么写，在Iterator 中Item似乎是必须存在的类型
    type Item = u32;
    fn next (&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}

fn main () {
    let mut counter = Counter::new();
    for item in counter.into_iter() {
        println!("{}", &item);
    }
}
```

## 使用自定义迭代器中其他 Iterator trait 方法

只要我们自己实现了 `Iterator`中的 `next`方法，就可以使用一些默认功能：

```rust
let sum: u32 = Counter::new()
        .zip(Counter::new().skip(1))
        .map(|(a, b)| a * b)
        .filter(|a| a % 3 == 0)
        .sum();

println!("{}", sum); //18
```

像里面的 `skip map filter sum` 都是可以直接用的。

注意 zip 只产生四对值；理论上第五对值 `(5, None)` 从未被产生，因为 `zip` 在任一输入迭代器返回 `None` 时也返回 `None`。

