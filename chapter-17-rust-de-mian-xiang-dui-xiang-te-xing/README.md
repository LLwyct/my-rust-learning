# Chapter 17 Rust的面向对象特性

在一些条件下，Rust 并不是面向对象语言：

* 没有继承、对象、封装、继承。

但是，Rust也可以说是一门面向对象语言：

* struct、enum 包含数据
* impl为上面的数据提供了方法
* 但带有方法的 struct 和 enum 在Rust中也不称为对象

在本章节中，我们会探索一些面向对象语言的特性，以及这些特性是如何体现在 Rust 中的。

接着会展示，如何在 Rust 中实现传统理解上的面向对象设计模式，并讨论这么实现与利用 Rust 自身的优势所实现的方案相比有什么区别、取舍。
