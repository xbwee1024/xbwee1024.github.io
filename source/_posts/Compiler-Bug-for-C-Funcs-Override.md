---
title: C++ 函数实现的异常覆盖
tags: 原创
top: false
cover: false
toc: true
mathjax: true
comments: true
date: 2020-12-25 18:09:59
categories:
    - C++
    - Android NDK
    - Compiler
summary:
---

`Android NDK` 有一些巨坑，编译链接没什么问题，运行时出错，而且非常难查。比如**声明有返回值的函数实现漏写`return`语句**， 会导致函数调用之后跑飞，`x86`不会有问题。最近遇到另外一个巨坑，采用静态链接第三方库，如果有同名函数，即使是函数签名不同，函数实现会被覆盖，也就是调用到的是第三方库实现，本地实现被覆盖掉, OMG~~~~

<!--more-->

编译环境：
**Host**： MacOS
**Target**: Android NDKr20
**ToolChain**: bazel
**Link**: Static Library

动态链接会报同名函数重定义错误~~ 这里讨论仅限于**静态链接**

本地函数声明如下，
```c++
#ifdef __cplusplus
namespace libfcc {
extern "C" {
#endif

LIBFCC_API
int I420ToNV12(const u8 *src_i420, u8 *dst_nv12, int width, int height);

#ifdef __cplusplus
}  // extern "C"
}  // namespace libfcc
#endif
```

Google `libyuv` 同名函数声明如下，
```c++
#ifdef __cplusplus
namespace libyuv {
extern "C" {
#endif

LIBYUV_API
int I420ToNV12(const uint8_t* src_y,
               int src_stride_y,
               const uint8_t* src_u,
               int src_stride_u,
               const uint8_t* src_v,
               int src_stride_v,
               uint8_t* dst_y,
               int dst_stride_y,
               uint8_t* dst_uv,
               int dst_stride_uv,
               int width,
               int height);

#ifdef __cplusplus
}  // extern "C"
}  // namespace libyuv
#endif
```

然后，神奇发生了，只要测试代码里面调用了该同名函数，运行时莫名崩溃，百思不得其解。

代码实现里添加log 没有打印，因为是在 benchmark 测试里面调用，刚开始怀疑是 benchmark 屏蔽了 log 输出。抽出来 `main` 函数测试也有问题，意识到是函数实现被覆盖了，注释掉本地函数实现，依然编译链接通过，没有任何错误，确定是编译链接问题。移除 `libyuv` 库依赖，一切恢复正常！！

这里函数签名不同也能被覆盖，是比较坑的地方！原因是因为用了 `extern "C"` 声明，也就是不会做 mangling 处理，导致编译器无法识别不同的函数签名。这也是 C 无法实现函数重载的原因。

那为何没有报函数重定义错呢？！

因为我虽然带着 `libyuv` 编译了，但是本地代码里面没有任何地方引用到`libyuv`，也就是没有 include `libyuv` 头文件。所以，编译器把我本地头文件的函数声明，与`libyuv` 的函数实现编译到一起了，既有函数声明，又有函数定义，所以不会有编译错误。

那为何编译器没有报函数调用参数缺少的`Warning`?!

NDK 编译，静态链接下不会有 Warning！甚至于添加 `-Werror`，也不会有 build error 信息。Why？


