# 17-3 面向对象设计模式的实现

这一节啊，其实很有启发。

整个这一章都在讲Rust与面向对象编程，但是这一节反而告诉你，不要被思维惯式禁锢了你的想法。

我们在熟悉了面向对象编程之后，自然而然的会使用类、对象、继承、多态等方案来幻想之后的编程。但是如今我们已经在学习一门新语言了，完全可以利用新语言的优势——所有权等来构筑新的思考方式。

但是作者还是很贴心的先使用面向对象的思维，使用继承这一套手段结合Rust为我们实现了一个 **博客发布过程** 的小例子。当我们惊叹其中手法的巧妙之时，作者突然笔锋一转，告诉你如果使用Rust的特性来实现这个例子，会在完成所有功能的情况下，更加完美的实现整个例子。

铺垫了这么多直接来看代码思路。

## 使用面向对象来实现博客发布过程

先做需求分析，一个博客有三个状态：

* 草稿，只能**添加内容**，无法**展示内容**，只能**送审**。
* 已审核，此时草稿已审核完毕，此时**无法添加内容**，也**无法展示内容**，只能继续**发布**。
* 已发布，此时是正式文档，不能**添加内容**，可以**展示内容**。

这个过程可以用一个有限状态机表示。

如果用传统的面向对象思考，很容易得出初步的解决方案，有一个**文章类**，它**上面有很多方法**：

* `new`，创建一个文章
* `add_text`，添加一些内容
* `content`，展示内容
* `request_review`，送审
* `approve`，发布

并且，这个文章类有两个成员：

* 状态
* 内容

然后创建子类继承于文章类：

* `Draft`，草稿
* `PendingReview`，审核中
* `Published`，已发布

那么这三个子类都要灵活的、或多或少实现，文章类的五个方法，只不过在不同状态下，具体逻辑不一样，比如草稿类调用送审，返回一个审核中类，但是审核中类送审，返回自身，因为它其实无法调用送审这个方法。

具体代码如下，或者去官网看一步步实现的逻辑，这里不赘述：

```rust
// lib.rs
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new () -> Post {
        Post {
            state: Some(Box::new(Draft {})),
            content: String::new(),
        }
    }
    pub fn add_text (&mut self, text: &str) {
        self.content.push_str(text);
    }
    // pub fn content (&self) -> &str {
    //     ""
    // }
    pub fn content (&self) -> &str {
        self.state.as_ref().unwrap().content(self)
    }
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review());
        }
    }
    pub fn approve (&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve());
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    fn approve(self: Box<Self>) -> Box<dyn State>;
    fn content<'a> (&self, post: &'a Post) -> &'a str {
        ""
    }
}

struct Draft {}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        Box::new(PendingReview {})
    }

    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}

struct PendingReview {}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
    fn approve(self: Box<Self>) -> Box<dyn State> {
        Box::new(Published{})
    }
}

struct Published {}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<dyn State> {
        self
    }
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
    fn content<'a> (&self, post: &'a Post) -> &'a str {
        &post.content
    }
}

// main.rs
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());

    post.request_review();
    assert_eq!("", post.content());

    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
}
```

其实如果你看懂了上面的实现代码，你会发现这和面向对象还是有一些小区别，因为这毕竟是Rust，没有原生支持各种面向对象操作。

我们这里是用**状态模式**，这一设计模式来实现面向对象。具体概念见[官网文档](https://kaisery.github.io/trpl-zh-cn/ch17-03-oo-design-patterns.html)。

## 将状态编码为不同类型

我们使用Rust的一些东西，来实现这一过程。

我们将展示如何稍微反思状态模式来进行一系列不同的权衡取舍。不同于完全封装状态和状态转移使得外部代码对其毫不知情，我们将状态编码进不同的类型。如此，Rust 的类型检查就会将任何在只能使用发布博文的地方使用草案博文的尝试变为编译时错误。

我们完全可以只用三种类型来代表，文章的三个状态：

* `Post`，已发布
* `PendingReviewPost`，审核中
* `DraftPost`，草稿

这样，只为这三种不同的类型实现各自的方法，这样，就能保证，当处于某一类型下，只能使用自己的方法。当你处于其他类型下时，是无法调用其他类型的方法的，这也借用了Rust强大的类型检查机制。比如，如果一个文章已发布，那么只能调用，查看内容 这一个方法。

以下是代码：

```rust
// lib.rs
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new () -> DraftPost {
        DraftPost { content: String::new() }
    }

    pub fn content (&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text (&mut self, text: &str) {
        self.content.push_str(text);
    }

    pub fn request_review (self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve (self) -> Post {
        Post {
            content: self.content
        }
    }
}
// main.rs
use blog::Post;
fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    // assert_eq!("", post.content()); 已无法调用

    let post = post.request_review();
    // assert_eq!("", post.content()); 已无法调用

    let post = post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
}
```

## 小节总结

其实你会发现这两种方案，对应的 `main.rs`中的内容很不同，前者的main.rs是很经典的面向对象写法，后者的你压根不能随便调用方法，因为类型不匹配会直接报错。

如果你是高手的话，你其实会发现，不管是哪种写法根本是无所谓的，不管是面向过程，还是面向对象都是一种思维，一种思考方式。你完全可以用Rust 模拟继承、封装、多态，也完全可以用纯面向过程的思路，完成第二种方式的代码，这只取决于你个人。

## 总结

阅读本章后，不管你是否认为 Rust 是一个面向对象语言，现在你都见识了 trait 对象是一个 Rust 中获取部分面向对象功能的方法。动态分发可以通过牺牲少量运行时性能来为你的代码提供一些灵活性。这些灵活性可以用来实现有助于代码可维护性的面向对象模式。Rust 也有像所有权这样不同于面向对象语言的功能。面向对象模式并不总是利用 Rust 优势的最好方式，但也是可用的选项。

接下来，让我们看看另一个提供了多样灵活性的 Rust 功能：模式。贯穿全书的模式, 我们已经和它们打过照面了，但并没有见识过它们的全部本领。让我们开始探索吧！

