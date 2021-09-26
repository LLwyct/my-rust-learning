# 9-1 panic! 与不可恢复的错误

突然有一天，代码出问题了，而你对此束手无策。对于这种情况，Rust 有 `panic!` 宏。当执行这个宏时，程序会打印出一个错误信息，展开并清理栈数据，然后接着退出。**出现这种情况的场景通常是检测到一些类型的 bug，而且程序员并不清楚该如何处理它。**

 发生 panic 时的展开或终止

> 当出现 panic 时，程序默认会开始 展开（unwinding），这意味着 Rust 会回溯栈并清理它遇到的每一个函数的数据，不过这个回溯并清理的过程有很多工作。另一种选择是直接 终止（abort），这会不清理数据就退出程序。那么程序所使用的内存需要由操作系统来清理。如果你需要项目的最终二进制文件越小越好，panic 时通过在 Cargo.toml 的 \[profile\] 部分增加 panic = 'abort'，可以由展开切换为终止。例如，如果你想要在release模式中 panic 时直接终止：

```text
[profile.release]
panic = 'abort'
```

## 简单的抛出一个panic

使用 `panic!` 宏

```typescript
fn main () {
    panic!("crash and burn");
}
```

控制台打印如下信息：

```text
(base) PS E:\project\hello_cargo> cargo run
   Compiling hello_cargo v0.1.0 (E:\project\hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.20s
     Running `target\debug\hello_cargo.exe`
thread 'main' panicked at 'crash and burn', src\main.rs:2:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
error: process didn't exit successfully: `target\debug\hello_cargo.exe` (exit code: 101)
```

错误信息指出错误发生在第2行 第5个字符处。但是我们经常可以引用他人的代码，最终错误会指向别人的 `panic!` 调用，而不是我们代码中最终导致panic的那一行。对于这种情况，我们可以使用 `panic!` 被调用的函数的 `backtrace` 来寻找代码中出问题的地方。下面我们会详细介绍 `backtrace` 是什么。

## 使用 panic! 的 backtrace

看一个因为我们的代码而引发别的库中的 `panic!` 例子：

```typescript
示例 9-1
fn main() {
    let v = vec![1, 2, 3];
    v[99];
}
```

```text
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27s
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', libcore/slice/mod.rs:2448:10
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

可以看到，由于我们的越界访问，造成了缓冲区溢出这个常见的bug。但是错误发生的位置指向了标准库中的 `libcore/slice/mod.rs`，这是 rust 中 slice 源码实现的地方。也是 panic 真实发生的地方。

并且错误信息还提示我们可以设置 `RUST_BACKTRACE` 环境变量来得到一个 `backtrace`。`backtrace` 是一个执行到目前位置所有被调用的函数的列表。Rust 的 `backtrace` 跟其他语言中的一样：阅读 `backtrace` 的关键是从头开始读直到发现你编写的文件。这就是问题的发源地。这一行往上是你的代码所调用的代码；往下则是调用你的代码的代码。这些行可能包含核心 Rust 代码，标准库代码或用到的 crate 代码。让我们将 `RUST_BACKTRACE` 环境变量设置为任何不是 `0` 的值来获取 `backtrace` 看看。

```text
RUST_BACKTRACE=1 cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', libcore/slice/mod.rs:2448:10
stack backtrace:
   0: std::sys::unix::backtrace::tracing::imp::unwind_backtrace
             at libstd/sys/unix/backtrace/tracing/gcc_s.rs:49
   1: std::sys_common::backtrace::print
             at libstd/sys_common/backtrace.rs:71
             at libstd/sys_common/backtrace.rs:59
   2: std::panicking::default_hook::{{closure}}
             at libstd/panicking.rs:211
   3: std::panicking::default_hook
             at libstd/panicking.rs:227
   4: <std::panicking::begin_panic::PanicPayload<A> as core::panic::BoxMeUp>::get
             at libstd/panicking.rs:476
   5: std::panicking::continue_panic_fmt
             at libstd/panicking.rs:390
   6: std::panicking::try::do_call
             at libstd/panicking.rs:325
   7: core::ptr::drop_in_place
             at libcore/panicking.rs:77
   8: core::ptr::drop_in_place
             at libcore/panicking.rs:59
   9: <usize as core::slice::SliceIndex<[T]>>::index
             at libcore/slice/mod.rs:2448
  10: core::slice::<impl core::ops::index::Index<I> for [T]>::index
             at libcore/slice/mod.rs:2316
  11: <alloc::vec::Vec<T> as core::ops::index::Index<I>>::index
             at liballoc/vec.rs:1653
  12: panic::main
             at src/main.rs:4
  13: std::rt::lang_start::{{closure}}
             at libstd/rt.rs:74
  14: std::panicking::try::do_call
             at libstd/rt.rs:59
             at libstd/panicking.rs:310
  15: macho_symbol_search
             at libpanic_unwind/lib.rs:102
  16: std::alloc::default_alloc_error_hook
             at libstd/panicking.rs:289
             at libstd/panic.rs:392
             at libstd/rt.rs:58
  17: std::rt::lang_start
             at libstd/rt.rs:74
  18: panic::main
```

这里有大量的输出！你实际看到的输出可能因不同的操作系统和 Rust 版本而有所不同。为了获取带有这些信息的 backtrace，必须启用 debug 标识。当不使用 --release 参数运行 cargo build 或 cargo run 时 debug 标识会默认启用，就像这里一样。

示例 9-2 的输出中，backtrace 的 12 行指向了我们项目中造成问题的行：src/main.rs 的第 4 行。如果你不希望程序 panic，第一个提到我们编写的代码行的位置是你应该开始调查的，以便查明是什么值如何在这个地方引起了 panic。在示例 9-1 中，我们故意编写会 panic 的代码来演示如何使用 backtrace，修复这个 panic 的方法就是不要尝试在一个只包含三个项的 vector 中请求索引是 100 的元素。当将来你的代码出现了 panic，你需要搞清楚在这特定的场景下代码中执行了什么操作和什么值导致了 panic，以及应当如何处理才能避免这个问题。

本章后面的小节 “panic! 还是不 panic!” 会再次回到 panic! 并讲解何时应该、何时不应该使用 panic! 来处理错误情况。接下来，我们来看看如何使用 Result 来从错误中恢复。

