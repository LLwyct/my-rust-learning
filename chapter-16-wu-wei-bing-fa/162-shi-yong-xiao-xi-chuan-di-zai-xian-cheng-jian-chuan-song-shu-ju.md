# 16-2 使用消息传递在线程间传送数据

在读完这一章后，我发现Rust 的多线程通信的核心思路和 Python 的多进程简直一模一样！

主要思路是，主进程创建一个Queue作为消息队列，然后把这个队列作为参数给其他子进程分发下去，子进程需要发送消息的时候，就把消息 `push` 到队列里，需要接受消息的进程就是用 `while(true)` 不断从队列 `get` 消息。当然，Rust 和 Python 一致的是对于一个队列实体，可以有多个子进程在上面调用 `push` 方法，但是只能有一个子进程对其调用 `get` 方法。当然，Python 不会对这一点进行检查，需要开发者自己注意，如果没有遵守这一约定，就会在运行时抛出异常。而Rust 会帮你做检查。

## 通道 Channel

> 一个日益流行的确保安全并发的方式是 消息传递（message passing），这里线程或 actor 通过发送包含数据的消息来相互沟通。这个思想来源于 Go 编程语言文档中 的口号：“不要通过共享内存来通讯；而是通过通讯来共享内存。”（“Do not communicate by sharing memory; instead, share memory by communicating.”）

Rust中一个**实现消息传递**的重要工具就是 **通道** （channel）。通道由 **发送者**（transmitter）和 **接收者**（receiver）组成。当发送者或接收者任意一方被丢弃时认为通道**关闭**（closed）了。

## 创建一个 Channel

```rust
use std::sync::mpsc;
fn main() {
    let (tx, rx) = mpsc::channel();
}
```

`mpsc::channel()` 会返回一个元组，分别是 **生产者** 和 **消费者**，其中 mpsc（multiple producer, single consumer）表示**可以有多个生产者，但只能有一个消费者**。

## 使用channel

* 生产者的 `send` 方法发送数据并返回一个 `Result<T, E>` ，当接收端关闭，就会返回一个错误。
* 接收者有两个方法：`recv` 和 `try_recv`。
  * `recv` 会阻塞线程直到收到一个值，一旦发送一个值，会返回一个 `Result<T, E>` ，当发送端关闭，就会返回一个错误。
  * `try_recv` 不会阻塞，它立刻返回一个 `Result<T, E>`：`Ok` 包含可用的信息，`Err` 代表此时没有消息，配合循环使用。

这里也需要 `move` 关键字把`tx`的所有权移入子线程。

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

## 通道与所有权转移

之前提到了，需要把生产者的所有权转移到子线程中。在子线程中，如果**使用`send` 函数发送给消费者的值也会跟着转移所有权**。

```rust
use std::thread;
use std::sync::mpsc;
fn main() {
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        println!("val is {}", val); // value used here after move
    });
    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

## 发送多个值并用迭代器接收

```rust
示例 16-10
use std::thread;
use std::sync::mpsc;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];
        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });
    for received in rx {
        println!("Got: {}", received);
    }
}
```

可以看到输出也是每打印一行暂停一秒，因为主线程中的 `for` 循环里并没有任何暂停或等待的代码，所以可以说主线程是在等待从新建线程中接收值。

## 通过克隆发送者来创建多个生产者

之前我们提到了`mpsc`是 _multiple producer, single consumer_ 的缩写。可以运用 `mpsc` 来扩展示例 16-10 中的代码来创建向同一接收者发送值的多个线程。这可以通过克隆通道的发送端来做到，如示例 16-11 所示：

```rust
let (tx, rx) = mpsc::channel();

let tx1 = tx.clone();
thread::spawn(move || {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("thread"),
    ];
    for val in vals {
        tx1.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

thread::spawn(move || {
    let vals = vec![
        String::from("more"),
        String::from("messages"),
        String::from("for"),
        String::from("you"),
    ];
    for val in vals {
        tx.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

for received in rx {
    println!("Got: {}", received);
}
```

现在我们见识过了通道如何工作，再看看另一种不同的并发方式吧。

