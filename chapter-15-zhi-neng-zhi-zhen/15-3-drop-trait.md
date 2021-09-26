# 15-3 Drop Trait

对智能指针第二重要的Trait是 `Drop` Trait，其作用是在值离开作用域时执行一些代码，比如释放资源。

其实 Drop 可以在任何类型上实现，不过它总是用于实现智能指针。比如`Box`自定义 Drop 用于释放堆空间。

## drop 方法

我们只需要给类型实现 `drop` 方法即可。`Drop` trait 包含在了preclude中，不需要引入。

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("CustomSmartPointers created.");
}
```

这段代码先打印 `other stuff`，说明释放的顺序是先进后出。

## 通过 `std::mem::drop` 提前丢弃值

`Drop` trait 的 `drop`方法的实现是在作用域结束后自动释放资源，`std::mem::drop` 方法可以手动释放资源。

该方法也是preclude的。

要注意的是不能主动使用Drop trait的drop方法，而是使用`std::mem::drop`。

```rust
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    c.drop(); // error!
    drop(c);  // ok!
    println!("CustomSmartPointer dropped before the end of main.");
}
```

