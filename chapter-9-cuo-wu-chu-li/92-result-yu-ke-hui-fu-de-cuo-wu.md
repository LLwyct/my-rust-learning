# 9-2 Result 与可恢复的错误

相比于上一节发生的严重错误，其实大多数错误并没有严重到需要程序立即停止执行。例如，因为打开一个不存在的文件失败，此时我们可以创建这个文件，而不是终止进程。

回忆一下第二章 “使用 Result 类型来处理潜在的错误” 部分中的那个 `Result` 枚举，它定义有如下两个成员，`Ok` 和 `Err`：

```typescript
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

## 一个简单的例子

我们在编写代码的时候，如果调用的是别人库里的函数，怎么知道返回的是不是一个 `Result` 呢？

* 查文档，[标准库API文档](https://doc.rust-lang.org/std/index.html)
* 随便设定一个值，看看报错是什么

```typescript
use std::fs::File;

fn main () {
    let f: u32 = File::open("hello.txt");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:4:18
  |
4 |     let f: u32 = File::open("hello.txt");
  |                  ^^^^^^^^^^^^^^^^^^^^^^^ expected u32, found enum
`std::result::Result`
  |
  = note: expected type `u32`
             found type `std::result::Result<std::fs::File, std::io::Error>`
```

可以看到报错的最后一行提示，发现了一个 `std::result::Result<std::fs::File, std::io::Error>` 类型。

这就告诉了我们，这个函数的返回值类型是什么。当执行成功时，返回值包括一个 `std::fs::File` **文件句柄** ，当失败时，E 的类型是 `std::io::Error`。

因此，我们需要根据返回值处理不同的逻辑。使用第六章的 match：

```typescript
use std::fs::File;

fn main () {
    let f = File::open("hello.txt");
    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("Problem opening the file: {:?}", error);
        },
    };
}
```

注意与 `Option` 枚举一样，`Result` 枚举和其成员也被导入到了 `preclude` 中，所以就不需要在 `match` 分支中的 `Ok` 和 `Err` 之前指定 `Result::`。

这里我们告诉 Rust 当结果是 `Ok` 时，返回 `Ok` 成员中的 `file` 值，然后将这个文件句柄赋值给变量 `f`。`match` 之后，我们可以利用这个文件句柄来进行读写。

`match` 的另一个分支处理从 `File::open` 得到 `Err` 值的情况。在这种情况下，我们选择调用 `panic!` 宏。如果当前目录没有一个叫做 `hello.txt` 的文件，当运行这段代码时会看到如下来自 `panic!` 宏的输出：

```text
thread 'main' panicked at 'Problem opening the file: Os { code: 2, kind: NotFound, message: "系统找不到指定的文件。" }', src\main.rs:8:13
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
error: process didn't exit successfully: `target\debug\hello_cargo.exe` (exit code: 101)
```

## 匹配不同的错误

我们可以修改一下上面的代码，以适配更加复杂的情况：

```typescript
use std::fs::File;
use std::io::ErrorKind;

fn main () {
    let f = File::open("hello.txt");
    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(file) => file,
                Err(e) => panic!("file not exit and created failed!"),
            },
            _ => panic!("problem opening the file"),
        },
    };
}
```

上面的代码虽然实现了，但是显得有一些冗余，包含了太多的match嵌套。第十三章会介绍闭包（closure）。`Result<T, E>` 有很多接受闭包的方法，并采用 `match` 表达式实现。一个更老练的 Rustacean 可能会这么写：

```typescript
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {:?}", error);
            })
        } else {
            panic!("Problem opening the file: {:?}", error);
        }
    });
}
```

虽然这段代码有着如示例 9-5 一样的行为，但并没有包含任何 match 表达式且更容易阅读。在阅读完第十三章后再回到这个例子，并查看标准库文档 `unwrap_or_else` 方法都做了什么操作。在处理错误时，还有很多这类方法可以消除大量嵌套的 match 表达式。

## 失败时 panic 的简写：unwrap 和 expect

match 能够胜任，不够显得有一些冗长。`Result<T, E>` 类型定义了很多辅助方法来处理各种情况。其中一种叫做 `unwrap` 。如果 `Result` 是 `Ok`，`unwrap` 会返回 `Ok` 中的值。如果 `Result` 是 `Err`，`unwrap` 会为我们调用 `panic!` 。

```typescript
use std::fs::File;

fn main () {
    let f = File::open("hello.txt").unwrap();
}
```

如果不存在 `hello.txt` ，我们将会看到一个 `unwrap` 调用 `panic!` 。

还有一个类似于 `unwrap` 的方法，`except` 允许我们选择 `panic!` 的错误信息。使用 `expect` 而不是 `unwrap` 并提供一个好的错误信息可以表明你的意图并更易于追踪 panic 的根源。

```typescript
use std::fs::File;

fn main () {
    let f = File::open("hello.txt").expect("open failed");
}
```

与 `unwrap` 不同的是，当返回值是 `Err` 时，**会抛出指定的错误信息**。

## 传播错误

当编写一些供他人使用的模块、包、函数的时候，我们可以选择让调用者知道哪里出了问题，并决定如何处理。因此我们尽量不要自己处理错误，而是把错误 **“传播（propagation）”** 出去。

下面编写一个函数实现这一点，这个函数的返回值比较特殊—— `Result<String, io::Error>` 。

```typescript
use std::io;
use std::fs::File;
use std::io::Read;
fn main () {
    let s = read_username_from_file().unwrap();
    println!("{}", s);
}
fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };
    let mut s = String::new();
    match f.read_to_string(&mut s) { /* 最后一行不需要return */
        Ok(_)/* 带构造的枚举好像必须加这个括号(_) */ => Ok(s),
        Err(e) => Err(e),
    }
}
```

调用这个函数的代码最终会得到一个包含用户名的 `Ok` 值，或者一个包含 `io::Error` 的 `Err` 值。我们无从得知调用者会如何处理这些值。我们只需要向上传播即可，让他们自己选择合适的方法。

**这种传播错误的模式在 Rust 是如此的常见，以至于 Rust 提供了 `?` 问号运算符来使其更易于处理。**

## 传播错误的简写：`?` 运算符

使用 `?` 运算符重写上面的代码。

```typescript
实例9-7
fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

Result 后的 `?` 运算符是指，当 `Result` 的类型为 `Ok` 时，整个表达式会返回 `Ok` 中的值，并继续向下执行；如果 `Result` 类型为 `Err` ，`Err` 中的值将作为整个函数的返回值，并结束当前上下文的执行，相当于 `return`。

> 我一开始在想，如果发生异常，不应该把整个Err\(error\) 返回吗？下面立刻回答了我的疑问。

与之前match版本不同的是，`?` 运算符会将错误值传递给 `from` 函数，它定义于标准库的 `From` trait 中，用来将错误从一种类型转换为另一种类型。当 `?` 运算符调用 `from` 函数时，**收到的错误类型被转换为由当前函数返回类型所指定的错误类型**。==这在当函数返回单个错误类型来代表所有可能失败的方式时很有用，即使其可能会因很多种原因失败。只要每一个错误类型都实现了 `from` 函数来定义如何将自身转换为返回的错误类型，`?` 运算符会自动处理这些转换。 ~~这句话也太抽象了吧？~~

在示例 9-7 的上下文中，`File::open` 调用结尾的 `?` 将会把 `Ok` 中的值返回给变量 `f`。如果出现了错误，`?` 运算符会提早返回整个函数并将一些 `Err` 值传播给调用者。同理也适用于 `read_to_string` 调用结尾的 ?。

`?` 运算符消除了大量样板代码并使得函数的实现更简单。我们甚至可以在 `?` 之后直接使用链式方法调用来进一步缩短代码，如示例 9-8 所示：

```typescript
fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}
```

其功能再一次与示例 9-6 和示例 9-7 保持一致，不过这是一个与众不同且更符合工程学\(ergonomic\)的写法。

说到编写这个函数的不同方法，甚至还有一个更短的写法：

```typescript
fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")
}
```

将文件读取到一个字符串是相当常见的操作，所以 Rust 提供了名为 `fs::read_to_string` 的函数，它会自动打开文件、新建一个 `String`、读取文件的内容，并将内容放入 `String`，接着返回它。当然，这样做就没有展示所有这些错误处理的机会了，所以我们最初就选择了艰苦的道路。

## ? 运算符可被用于返回 Result 的函数

`?` 运算符应该被用于返回值为 Result 的函数。

如果在 `main` 函数中使用 `?` 运算符会报错，因为 `main` 函数的返回值是 `()`。

```text
cannot use the `?` operator in a function that returns `()`
```

错误指出只能在返回 `Result` 或者其它实现了 `std::ops::Try` 的类型的函数中使用 `?` 运算符。当你期望在不返回 `Result` 的函数中调用其他返回 `Result` 的函数时使用 `?` 的话，有两种方法修复这个问题。一种技巧是将函数返回值类型修改为 `Result<T, E>`，如果没有其它限制阻止你这么做的话。另一种技巧是通过合适的方法使用 `match` 或 `Result` 的方法之一来处理 `Result<T, E>`。

`main` 函数是特殊的，其必须返回什么类型是有限制的。`main` 函数的一个有效的返回值是 `()`，同时出于方便，另一个有效的返回值是 `Result<T, E>`，如下所示：

```typescript
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;

    Ok(())
}
```

`Box<dyn Error>` 被称为 “trait 对象”（“trait object”），第十七章 “为使用不同类型的值而设计的 trait 对象” 部分会做介绍。目前可以理解 `Box<dyn Error>` 为使用 `?` 时 main 允许返回的 “任何类型的错误”。

现在我们讨论过了调用 `panic!` 或返回 `Result` 的细节，是时候回到他们各自适合哪些场景的话题了。

