# 4-3 slice

官方文档中这一部门讲的太烂了，我直接用我的理解来描述。

## Slices 类型

上一章我们提到了借用与引用，可以绕过所有权，这一章我们聊另一个没有所有权的数据类型是 `slice`。`slice` 允许你引用集合中一段连续的元素序列，而不用引用整个集合。

考虑一个场景，我们需要获得一段话的第一个单词。根据我们以往的编程经验，也就是我们要找到第一个空格，并对字符串进行分割。

Slice 可以获得字符串的一部分值的引用。

**因为 `Slice` 的本质是引用，本身并不在堆上创建额外的真实数据**，因此一举两得：

* 首先，**可以绕过所有权**
* 其次，**可以起到分割的效果**

写法如下：

```javascript
let s = String::from("hello world");
let t = &s[start_idx .. end_idx]; // 不包括结尾
let brorrow         = &s; // 普通引用
let index_to_end     = &s[3..]; // 从index到结束
let start_to_index     = &s[..8]; // 从开始到index
let complete_string = &s[..]; // 获得整个的引用
let hello             = &s[0..5]; // h 到 o的引用
let world             = &s[6..11]; // w 到 d的引用
```

从下图可以看出，`s` 作为原本的 `String` ，`world` 只是创建了一个指向s一部分的引用。 ![&#x5728;&#x8FD9;&#x91CC;&#x63D2;&#x5165;&#x56FE;&#x7247;&#x63CF;&#x8FF0;](https://img-blog.csdnimg.cn/9c488f1ff5fa404e8a265011096539de.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAc3dhbGxvd2JsYW5r,size_10,color_FFFFFF,t_70,g_se,x_16)

> 注意：字符串 slice range 的索引必须位于有效的 UTF-8 字符边界内，如果尝试从一个多字节字符的中间位置创建字符串 slice，则程序将会因错误而退出。出于介绍字符串 slice 的目的，本部分假设只使用 ASCII 字符集；第八章的 “使用字符串存储 UTF-8 编码的文本” 部分会更加全面的讨论 UTF-8 处理问题。

有了以上补充，我们可以创建一个 `first_world` 函数。

```javascript
fn main () {
    let mut s = String::from("hello world");
    let word = first_word(&s);
    s.clear(); // failure
}

fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[..i];
        }
    }
    return &s[..];
}
```

思考一下主函数第三行为什么失败？是因为对于`String s` 目前有一个不可变引用 `word`，然而代码想尝试删除其中的数据，这是不允许的。**回忆一下借用规则，当拥有某值的不可变引用时，就不能再获取一个可变引用。**

## 震惊！！字符串字面值其实也是 slice

```javascript
let s = "hello world";
```

是不是以为字符串字面值实际指向的是堆上的真实值？其实不是，这里的 `s` 其实是 `"hello world"` 的 slice。

这里的 `s` 的类型是 `&str`。它是一个指向二进制程序特定位置的 `slice`。**这也就是为什么字符串字面值是不可变的**；`&str` 是一个不可变引用。

## 字符串 slice 作为参数

对函数 `first_word` 的函数签名进行改进，因为这样甚至可以直接传入字符串字面量：

```javascript
fn first_word(s: &str) -> &str

fn main() {
    let origin_str = "hello world";
    let string_str = String::from("hello world");
    let _ = first_word(origin_str);
    let _ = first_word(&string_str[..]);
}
```

## 其他类型的 slice

`slice` 当然不值适用于字符串，也有更加通用的类型。

```javascript
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];
```

这个 `slice` 的类型是 `&[i32]`。它跟字符串 `slice` 的工作方式一样，通过存储第一个集合元素的引用和一个集合总长度。你可以对其他所有集合使用这类 `slice`。第八章讲到 `vector` 时会详细讨论这些集合。

## 总结

所有权、借用和 slice 这些概念让 Rust 程序在编译时确保内存安全。Rust 语言提供了跟其他系统编程语言相同的方式来控制你使用的内存，但拥有数据所有者在离开作用域后自动清除其数据的功能意味着你无须额外编写和调试相关的控制代码。

所有权系统影响了 Rust 中很多其他部分的工作方式，所以我们还会继续讲到这些概念，这将贯穿本书的余下内容。让我们开始第五章，来看看如何将多份数据组合进一个 struct 中。

