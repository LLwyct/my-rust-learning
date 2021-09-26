---
description: 常用循环
---

# 第三章 循环 loop while for range

## 1 loop 循环

```javascript
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;
    }
};
```

下面这样写也是可以的，只不过这样break语句后面的话也能运行吗？

```javascript
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break
        counter * 2;
    }
};
```

这样就又不行了，可见分号在rust中扮演着重要的角色，如果 break 后有分号，则视为结束，如果没有则还能再跟一个表达式表示返回值。

```javascript
if counter == 10 {
    break;
    counter * 2;
}
```

这样又不行了。

```javascript
if counter == 10 {
    break
    counter += 1;
}
```

## 2 while 循环

这个很简单，没什么可说的

```javascript
/**
* ? while 循环
*/
let mut number = 3;

while number != 0 {
    number -= 1;
}

println!("while result: {}", number);
```

## 3 for 循环与 Range

RUST 中的 `for` 循环有一些不太一样，它可以自动遍历一个生成器来遍历可迭代数据结构。

```javascript
// ? while 效率低，需要检查条件
// ? 迭代器循环
let a = [1, 2, 4, 3, 6];
for item in a.iter() {
    println!("value in a is {}", item);
}
```

使用 `Range` 创建一个可迭代的区间

```javascript
// Range[start, end) 生成器 用两个点表示 .. 
// rev方法表示逆序
for i in (0..5).rev() {
    println!("value in a is {}", a[i]);
    // value in a is 6
    // value in a is 3
    // value in a is 4
    // value in a is 2
    // value in a is 1
}
```

