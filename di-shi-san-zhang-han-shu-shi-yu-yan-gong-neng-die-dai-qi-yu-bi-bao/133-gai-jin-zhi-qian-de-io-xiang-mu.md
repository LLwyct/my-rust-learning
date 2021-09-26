# 13-3 改进之前的IO项目

在第十二章我们创建了一个基于命令行的，查询文件内容的小工具，今天来继续完善提高。

## 使用迭代器来获取命令行参数

由于获取命令行参数的函数 `env::args()` 的返回值正好是一个迭代器，因此我们直接用它来实现。

主函数：

```rust
let config = Config::new(env::args()).unwrap_or_else(|err| {
    eprintln!("参数解析失败: {}", err);
    process::exit(1);
});
原代码
let args: Vec<String> = env::args().collect();
let config = Config::new(&args).unwrap_or_else(|err| {
    eprintln!("参数解析失败: {}", err);
    process::exit(1);
});
```

`config` 的构造函数：

```rust
pub fn new (mut args: env::Args) -> Result<Config, &'static str> {
    if args.len() < 3 {
        return Err("未输入参数");
    }
    args.next();
    let query = match args.next() {
        Some(v) => v,
        None => return Err("无法找到查询字符串参数"),
    };
    let filename = match args.next() {
        Some(v) => v,
        None => return Err("无法找到文件名参数"),

    };
    ......
}

// 原先的代码
pub fn new (args: &[String]) -> Result<Config, &'static str> {
    if args.len() < 3 {
        return Err("未输入参数");
    }
    let query = args[1].clone();
    let filename = args[2].clone();
    ......
}
```

`search_sensitive` 函数：

```rust
pub fn search_sensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    contents.lines().filter(|line| line.contains(query)).collect()
}

// 原先的函数
pub fn search_sensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
             results.push(line);
        }
    }
    results
}
```

`search_insensitive` 函数：

```rust
pub fn search_insensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    contents
    .lines()
    .filter(
        |line| line.to_lowercase().contains(&query.to_lowercase())
    )
    .collect()
}

// 原先的函数
pub fn search_sensitive<'a> (query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();
    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }
    results
}
```

