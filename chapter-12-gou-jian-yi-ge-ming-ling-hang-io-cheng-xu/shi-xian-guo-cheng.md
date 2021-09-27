# 实现过程

## 创建一个新的项目

```bash
cargo new minigrep
```

## 完成基本的功能

在项目根目录下创建 poem.txt

```text
I'm Nobody! Who are you?
Are you – Nobody – too?
Then there's a pair of us!
Don't tell! they'd advertise – you know!

How dreary – to be – Somebody!
How public – like a Frog –
To tell one's name – the livelong June –
To an admiring Bog!
```

底下这个程序完成了基本的功能，不再赘述，稍微学过编程的都能看懂。

1. 从命令行读取参数
2. 按照参数打开文件并读取文件内容

```rust
use std::env;
use std::fs;

fn main() {
    let args: Vec<String> = env::args().collect();

    let query = &args[1];
    let filename = &args[2];

    let contents = fs::read_to_string(filename).expect("读取文件时发生错误");
    println!("{}\n", contents);
}
```

## 第一次重构：对代码结构整理

上面的代码，最多称为一段代码，离真正的程序还差的远，忽略了许多规范，于是对其进行重构。

我们认为，query 和 filename 应该是关联的，因此我们把它们作为一个结构体包裹起来，并且把获取参数这部分作为一个函数封装一下，这个函数的返回值就是一个 `Config`，那么这个函数可以认为是 `config`的一个构造函数，因此我们再用 `impl` 对其进行重写。

在构造结构体时，要注意 String 类型的所有权转移，因此使用 `clone` 函数进行复制。

```rust
use std::env;
use std::fs;

fn main() {
    let args: Vec<String> = env::args().collect();
    let config = Config::new(&args);
    let contents = fs::read_to_string(config.filename)
        .expect("读取文件时发生错误");
    println!("{}\n", contents);
}

struct Config {
    query: String,
    filename: String,
}

impl Config {
    fn new (args: &[String]) -> Config {
        let query = args[1].clone();
        let filename = args[2].clone();
        Config {
            query,
            filename
        }
    }
}
```

## 第二次重构：对错误进行处理

看一下对new函数的重写，当参数列表的长度小于3时，直接报错。

```rust
fn new (args: &[String]) -> Result<Config, &'static str> {
    if args.len() < 3 {
        panic!("未输入参数");
    }
    let query = args[1].clone();
    let filename = args[2].clone();
    Config {
        query,
        filename
    }
}
```

但是这么一写，当发生错误时，命令行会打印很多对用户的无用信息，因此我们用之前学过的 `Result` 重构一下。

```rust
use std::env;
use std::fs;
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();
    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("解析参数时出错：{}", err);
        process::exit(1);
    });
    let contents = fs::read_to_string(config.filename)
        .expect("读取文件时发生错误");
    println!("{}\n", contents);
}

struct Config {
    query: String,
    filename: String,
}

impl Config {
    fn new (args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("未输入参数");
        }
        let query = args[1].clone();
        let filename = args[2].clone();
        Ok(
            Config { query, filename }
        )
    }
}
```

这里其实是一个匿名函数。当发生错误时，调用 `unwrap_or_else` 中的匿名函数。并且通过 process 标准库，来退出程序。这样就只会打印错误内容。

```rust
|err| {
    println!("解析参数时出错：{}", err);
    process::exit(1);
}
```

## 第三次重构：模块化

**二进制项目的关注分离**

main 函数负责多个任务的组织问题在许多二进制项目中很常见。所以 Rust 社区开发出一类在 main 函数开始变得庞大时进行二进制程序的**关注分离的指导性过程**。

**社区希望开发人员在编写Binary程序时，注重如下步骤**：

* 将程序拆分成 `main.rs` 和 `lib.rs` 并将程序的逻辑放入 `lib.rs` 中。
* 当命令行解析逻辑比较小时，可以保留在 `main.rs` 中。
* 当命令行解析开始变得复杂时，也同样将其从 `main.rs` 提取到 `lib.rs` 中。

经过这些过程之后保留在 `main` 函数中的责任应该被限制为：

* 使用参数值调用命令行解析逻辑
* 设置任何其他的配置
* 调用 `lib.rs` 中的 `run` 函数
* 如果 `run` 返回错误，则处理这个错误

这个模式的一切就是为了关注分离：`main.rs` 处理程序运行，而 `lib.rs` 处理所有的真正的任务逻辑。因为不能直接测试 `main` 函数，这个结构通过将所有的程序逻辑移动到 `lib.rs` 的函数中使得我们可以测试他们。仅仅保留在 `main.rs` 中的代码将足够小以便阅读就可以验证其正确性。让我们遵循这些步骤来重构程序。

创建 `lib.rs`，并新增一个 `run` 函数，令run 函数的返回值为 `Result<(), Box<dyn Error>>`，其中的Box 是一个实现了 `Error` trait 的类型。

```rust
use std::fs;
use std::error::Error;

pub fn run (config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;
    println!("{}\n", contents);
    Ok(())
}

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new (args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("未输入参数");
        }
        let query = args[1].clone();
        let filename = args[2].clone();
        Ok(
            Config {
                query,
                filename
            }
        )
    }
}
```

再看看main.rs，这里处理 run 的错误 和 config 构造函数的 错误，用了不同的方法，因为当run函数成功时，我们不关注它的返回值，是一个空元组，我们只关心错误时的返回值。

```rust
use std::env;
use std::process;
use minigrep::Config;

fn main() {
    let args: Vec<String> = env::args().collect();
    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("解析参数时出错：{}", err);
        process::exit(1);
    });
    if let Err(e) = minigrep::run(config) {
        println!("运行时错误：{}", e);
        process::exit(1);
    }
}
```

## 使用TDD 测试驱动开发

在这一部分，我们将遵循测试驱动开发（Test Driven Development, TDD）的模式来逐步增加 minigrep 的搜索逻辑。这是一个软件开发技术，它遵循如下步骤：

* 编写一个失败的测试，并运行它以确保它失败的原因是你所期望的。
* 编写或修改足够的代码来使新的测试通过。
* 重构刚刚增加或修改的代码，并确保测试仍然能通过。
* 从步骤 1 开始重复！

这只是众多编写软件的方法之一，不过 TDD 有助于驱动代码的设计。在编写能使测试通过的代码之前编写测试有助于在开发过程中保持高测试覆盖率。

我们将测试驱动实现实际在文件内容中搜索查询字符串并返回匹配的行示例的功能。我们将在一个叫做 `search` 的函数中增加这些功能。

先编写一个测试模块

```rust
pub fn search<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    vec![]
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_result () {
        let query = "duct";
        let contents = "\
Rust:
safe, fast and productive.
I like it.";
        assert_eq!(vec!["safe, fast and productive."], search(query, contents));
    }
}
```

我们先在函数 `one_result` 编写测试用例，手写一个 `contents`，包含三行内容。定义查询内容 `duct` 为 “query”。

我们希望通过 `search` 函数之后，返回的值，等于 `contents`的第二行，而且事实本该如此。但是此时 `search` 函数的返回值是空向量。

在运行 `cargo test` 后自然是不对的。

现在编写 `search` 函数，很简单，不解释，主要这里要**注意一下引用的生命周期注解**。

```rust
pub fn search<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }
    results
}
```

再运行 `cargo test`，测试成功。此时修改 `run` 函数。

```rust
pub fn run (config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;
    for line in search(&config.query, &contents) {
        println!("{}", line);
    }
    Ok(())
}
```

最后测试我们的代码

```bash
cargo run body poem.txt
cargo run 123123 poem.txt
```

都没有问题。

目前为止的完整代码：

{% tabs %}
{% tab title="main.rs" %}
```rust
use minigrep::Config;
use std::env;
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();
    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("解析参数时出错：{}", err);
        process::exit(1);
    });
    if let Err(e) = minigrep::run(config) {
        println!("运行时错误：{}", e);
        process::exit(1);
    }
}
```
{% endtab %}

{% tab title="lib.rs" %}
```rust
use std::fs;
use std::error::Error;

pub fn run (config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;
    for line in search(&config.query, &contents) {
        println!("{}", line);
    }
    Ok(())
}

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new (args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("未输入参数");
        }
        let query = args[1].clone();
        let filename = args[2].clone();
        Ok(
            Config {
                query,
                filename
            }
        )
    }
}

pub fn search<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }
    results
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_result () {
        let query = "duct";
        let contents = "\
Rust:
safe, fast and productive.
I like it.";
        assert_eq!(vec!["safe, fast and productive."], search(query, contents));
    }
}
```
{% endtab %}
{% endtabs %}

## 处理环境变量

我希望引入一个功能，为查询提供是否区分大小写的能力，并且是否区分不来自于用户的输入参数，而是环境变量，让我们开始吧！

首先修改原始的 `search` 函数，改为 `search_sensitive` ，函数内容不变，因为原本就是要区分大小写的。

我们添加一个TDD模块：

```rust
#[test]
fn case_insensitive () {
    let query = "rust";
    let contents = "\
Rust:
safe, fast and productive.
I like it.
Duct tape.
Trust me!";
    assert_eq!(vec!["Rust:", "Trust me!"], search_insensitive(query, contents));
}
```

再定义我们的 `search_insensitive` 函数。

```rust
pub fn search_insensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    let query = query.to_lowercase();
    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }
    results
}
```

修改一下 `run` 函数的逻辑，根据 `config` 里面的 `case_sensitive` 字段来选择使用哪个函数。

```rust
pub fn run (config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;
    let results = if config.case_sensitive {
        search_sensitive(&config.query, &contents)
    } else {
        search_insensitive(&config.query, &contents)
    };
    for line in results {
        println!("{}", line);
    }
    Ok(())
}
```

看一下 `config` 的构造函数，为 `config` 结构体增加新字段，并添加从环境变量提取该字段值的逻辑。`is_err` 是指，当返回的 `Result` 类型的值是 `Ok` 时返回 `false`，是`Err`时返回 `true`，因为逻辑在这里只关心环境变量有没有 `"CASE_INSENSITIVE"`，并不关心是什么。

```rust
pub fn new (args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("未输入参数");
        }
        let query = args[1].clone();
        let filename = args[2].clone();
        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();
        Ok(
            Config {
                query,
                filename,
                case_sensitive,
            }
        )
    }
```

并且从头到尾都没有修改过 `main.rs` 这就是模块化的好处。

## 

{% hint style="info" %}
下面是windows OS下的切换当前环境变量的命令，只对当前控制台有效。

```bash
$env:CASE_INSENSITIVE=1
```
{% endhint %}

## 标准错误与标准输出

有时候我们希望输出结果到一个文件，但是如果发生了错误，错误信息也会输出到文件，但是我只希望文件保存正确的结果，这时候可以使用标准错误。

```text
cargo run > output.txt
```

这句话的意思是输出流从控制台重定向到文本。很明显，这个命令没有参数，会报错到 output.txt。

可以使用标准错误，`eprintln!()`。

## 完整代码

最后看一下完整代码：

```rust
// main.rs
use minigrep::Config;
use std::env;
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();
    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("参数解析失败: {}", err);
        process::exit(1);
    });
    if let Err(e) = minigrep::run(config) {
        eprintln!("运行失败: {}", e);
        process::exit(1);
    }
}
```

```rust
// lib.rs
use std::fs;
use std::env;
use std::error::Error;

pub fn run (config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;
    let results = if config.case_sensitive {
        search_sensitive(&config.query, &contents)
    } else {
        search_insensitive(&config.query, &contents)
    };
    for line in results {
        println!("{}", line);
    }
    Ok(())
}

pub struct Config {
    pub query: String,
    pub filename: String,
    pub case_sensitive: bool,
}

impl Config {
    pub fn new (args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("未输入参数");
        }
        let query = args[1].clone();
        let filename = args[2].clone();
        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();
        Ok(
            Config {
                query,
                filename,
                case_sensitive,
            }
        )
    }
}

pub fn search_sensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }
    results
}

pub fn search_insensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    let query = query.to_lowercase();
    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }
    results
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn case_sensitive () {
        let query = "duct";
        let contents = "\
Rust:
safe, fast and productive.
I like it.
Duct tape.";
        assert_eq!(vec!["safe, fast and productive."], search_sensitive(query, contents));
        // assert_eq!(vec!["safe, fast and productive.", "Duct tape."], search_sensitive(query, contents));
    }
    #[test]
    fn case_insensitive () {
        let query = "rust";
        let contents = "\
Rust:
safe, fast and productive.
I like it.
Duct tape.
Trust me!";
        assert_eq!(vec!["Rust:", "Trust me!"], search_insensitive(query, contents));
    }
}
```

