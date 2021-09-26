---
description: 如何在不安装VS的情况下快速安装Rust
---

# 第一章 如何快速安装Rust

最近想学习一下rust语言，于是打算安装一下。

在windows平台下载好安装包后，一切照常的打开安装包，提示如下：

```bash
Rust Visual C++ prerequisites

Rust requires the Microsoft C++ build tools for Visual Studio 2013 or
later, but they don't seem to be installed.

The easiest way to acquire the build tools is by installing Microsoft
Visual C++ Build Tools 2019 which provides just the Visual C++ build
tools:

  https://visualstudio.microsoft.com/visual-cpp-build-tools/

Please ensure the Windows 10 SDK and the English language pack components
are included when installing the Visual C++ Build Tools.

Alternately, you can install Visual Studio 2019, Visual Studio 2017,
Visual Studio 2015, or Visual Studio 2013 and during install select
the "C++ tools":

  https://visualstudio.microsoft.com/downloads/

Install the C++ build tools before proceeding.

If you will be targeting the GNU ABI or otherwise know what you are
doing then it is fine to continue installation without the build
tools, but otherwise, install the C++ build tools before proceeding.

Continue? (y/N)
```

大致的意思就是说，RUST 是依赖与 Visual Studio的，需要安装里面的windows 10 SDK 部分，大致关系和Flutter 和 Android Studio 比较像。

所以很多人就不得不去下载VS，但是众所周知，VS一下就是好几个G出去了，如果仅仅为了学习RUST而去下载这么多东西是不值得的（因为我实验室的电脑已经很破旧了，已经非常卡了，我不会为了学习rust和影响我日常的使用）。

于是我在Stack Overflow发现了一个很牛逼的方法，只需要下载几百兆的东西即可。

ocanis 在 [Stack Overflow](https://stackoverflow.com/questions/55603111/unable-to-compile-rust-hello-world-on-windows-linker-link-exe-not-found) 的回答如下：

![&#x5177;&#x4F53;&#x56DE;&#x7B54;](https://img-blog.csdnimg.cn/9270b7060f0a447baf910077c6bf5bb6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N3YWxsb3dibGFuaw==,size_16,color_FFFFFF,t_70)

