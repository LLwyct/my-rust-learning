# 6-2 match 控制流运算符

## match控制流运算符

Rust 有一个叫做 `match` 的极为强大的控制流运算符，它允许我们将一个值与一系列模式比较，并根据相匹配的模式执行相应代码吗。

模式可由 **字面值**、**变量**、**通配符**和**许多其他内容**构成；第十八掌会涉及所有种类的模式，以及作用。

`match` 的强大来源于模式的表现力，以及编译器检查，**确保所有可能的内容都做了处理**。

## 一个简单的🌰

```typescript
enum Coin {
    Penny,
    Nickel
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => {
            println!("This is 5 cent");
            5
        },
    }
}
```

一个很通用的例子，对硬币进行分类，对于简单的情况直接在箭头后返回值，对于复杂的例子可以使用大括号。

## 绑定值的match

上一节提到了枚举和泛型，我知道了 `Option<i32>` 和 `i32` 是两个不同的类型，无法比较或运算，那么如何提取泛型里的值呢？

匹配分支的另一个功能是可以绑定匹配的match的部分值。这也就是如何从枚举成员中提取值的。

改进之前的例子，在所有的硬币中，只有25美分的硬币是由不同州发行的，其余硬币都是通用的。

我假设只有加利福尼亚州和华盛顿两个地方，然后给25美分绑定州的能力，这样如果给函数 `value_in_cents` 传进去的是25美分就会打印对应的州。

```typescript
#[derive(Debug)]
enum UsState {
    California,
    Washington,
}

enum Coin {
    Penny,  // 1 cent
    Nickel, // 5 cent
    Dime,   // 10 cent
    Quarter(UsState),// 25 cent
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("这枚硬币来自于 {:?} 州", state);
            25
        }
    }
}

fn main () {
    let cent = Coin::Quarter(UsState::California);
    value_in_cents(cent);
}

(base) E:\hello_cargo> 这枚硬币来自于 California 州
```

## 匹配 `Option <T>`

匹配 `Option<T>` 和 上面硬币的例子很像，因为 `Option` 其实本身就是一个 `enum`。

比如我们想要编写一个函数，它获取一个 `Option<i32>` ，如果其中含有一个值，将其加一。如果其中没有值，函数应该返回 `None` 值，而不尝试执行任何操作。

```typescript
fn main () {
    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);
}

fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i+1),
    }
}
```

比较 `Some(5)` 和 `Some(i)` ，它们匹配吗？当然匹配！它们是相同的成员。`i` 绑定了 `Some` 中包含的值，所以 `i` 的值是 `5`。接着匹配分支的代码被执行，所以我们将 `i` 的值加一并返回一个含有值 `6` 的新 `Some`。

将 `match` 与枚举相结合在很多场景中都是有用的。你会在 Rust 代码中看到很多这样的模式：`match` 一个枚举，绑定其中的值到一个变量，接着根据其值执行代码。这在一开始有点复杂，不过一旦习惯了，你会希望所有语言都拥有它！这一直是用户的最爱。

## 匹配是穷尽的！通配符?!

这里我用我的话来解释一下，就是Rust的机制是如果你使用 `match` 匹配，编译器会检查穷尽所有的可能，比如你无法仅仅只使用 `Some` 来匹配 `Option<T>`，因为 `Option` 游有可能是 `None`。 再比如，`match` 后跟枚举类型，那么你就必须指明该枚举所有类型的处理。

**当然可以使用通配符。**

回顾之前的硬币例子，我对核心函数进行修改。我只希望打印25美分的硬币类型，其他的都按照 0美分来处理。我可以使用 下划线 `_` 作为通配符。

```typescript
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Quarter(state) => {
            println!("这枚硬币来自于 {:?} 州", state);
            25
        },
        _ => 0
    }
}
```

如果没有通配符的话编译器会提示，你没有覆盖所有类型。

> `_ => ()` 表示什么都不做。

然而，如果只关心 一个 情况的话使用 match 显得有点啰嗦。为此 Rust 提供了 `if let`。

