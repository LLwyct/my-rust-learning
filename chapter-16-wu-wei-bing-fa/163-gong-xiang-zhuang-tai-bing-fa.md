# 16-3 共享状态并发

上一节提到了，“不要通过共享内存来通信，要用通信来共享内存”。

虽说如此，我们还是来看一下，如何在Rust 中使用共享内存。

本质上，使用 Channel 相当于 单所有权；共享内存相当于 多所有权，多个线程可以同时访问相同的内存位置。第十五章介绍的智能指针使多所有权成为可能。

## 使用互斥器 Mutex

使用**互斥器**（Mutex，mutual exclusion），同一时刻，它只允许一个线程访问数据。

线程想要访问互斥器中的数据，必须先获取互斥器的 **锁**（lock）。这个锁是互斥器数据结构的一部分，记录谁有数据的排他访问权。

因此，互斥器通过锁系统来保护数据安全。

**互斥器十分难用，它有必须的规则**：

* 在使用数据前先尝试获得锁
* 处理完互斥器保护的数据后，必须解锁，让其他线程可以获取锁。

## `Mutex<T>` 的 API

通过 `Mutex::new(data)` 来创建 `Mutex<T>`，`Mutex<T>` **本质上是一个智能指针**。

访问数据前，通过 `lock` 方法获取锁

* `lock` 方法会阻塞线程
* `lock` 方法可能会失败
* `lock` 方法返回的是 `LockResult<MutexGuard<'_, T>>`，`MutexGuard<'_, T>`也是一个智能指针，实现了 `Deref` 和 `Drop`  。

```typescript
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }

    println!("m = {:?}", m);
}
```

先用 new 创建一个互斥器 `mutex<T>`，在子作用域中调用互斥器的`lock`方法获取锁，获取的过程会阻塞当前进程，如果锁被别的进程占用，就会一直阻塞。当另一个进程获取锁但是触发了`panic`，那么`lock`会调用失败，返回`Err`，所以调用 `unwrap`方法。

现在获取了锁，经过 `unwrap` 的返回值就是一个可变引用。因为 `lock` 的返回值实际是一个`LockResult<MutexGuard<'_, T>>`，其中的 `MutexGuard<'_, T>` 是一个智能指针，实现了`Deref` 和 `Drop` Trait。所以它可以通过deref直接指向内部的数据，并且在离开作用域后自动释放锁。

## 在线程间共享 `Mutex<T>`

编写一个例子：在主进程中创建十个线程，给某个值加一。

```typescript
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

这段代码会编译失败，`互斥器 counter 被移动进了闭包`。意思就是在第一次循环的时候 `counter` 已经move到了闭包中，那么后面的循环已经无法获得 `counter` 这个变量了。所以 Rust 告诉我们不能将 `counter` 锁的所有权移动到多个线程中。让我们通过一个第十五章讨论过的多所有权手段来修复这个编译错误。

### 多线程和多所有权

在第十五章中，通过智能指针 `Rc<T>` 以便拥有多所有者。修改一部分代码：

```typescript
use std::rc::Rc;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];
    for _ in 0..10 {
        let counter = Rc::clone(&counter);
```

还是报错，编译器也告诉了我们原因 `the trait bound Send is not satisfied`。下一部分会讲到 `Send`：这是确保所使用的类型可以用于并发环境的 trait 之一。

之前也说过了，Rc智能指针不能用于并发环境。`Rc<T>` 并没有使用任何并发原语，来确保改变计数的操作不会被其他线程打断。在计数出错时可能会导致诡异的 bug，比如可能会造成内存泄漏，或在使用结束之前就丢弃一个值。我们所需要的是一个完全类似 `Rc<T>`，又以一种线程安全的方式改变引用计数的类型。

### 原子引用计数 `Arc<T>`

所幸 `Arc<T>` 正是 这么一个类似 `Rc<T>` 并可以安全的用于并发环境的类型。字母 “a” 代表 原子性（atomic），所以这是一个原子引用计数（atomically reference counted）类型。

回到之前的例子：`Arc<T>` 和 `Rc<T>` 有着相同的 API，所以修改程序中的 use 行和 new 调用

```typescript
use std::sync::{Mutex, Arc};
use std::thread;
use std::rc::Rc;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

## `RefCell<T>`/`Rc<T>` 与 `Mutex<T>`/`Arc<T>` 的相似性

* `Mutex<T>` 提供了内部可变性，和Cell家族一样
* 使用`RefCell<T>` 来改变 `Rc<T>`中的内容
* 使用`Mutex<T>` 来改变 `Arc<T>`中的内容
* 注意：`Mutex<T>` 有死锁风险

 接下来，为了丰富本章的内容，让我们讨论一下 `Send`和 `Sync` trait 以及如何对自定义类型使用他们。

