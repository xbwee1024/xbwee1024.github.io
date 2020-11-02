---
title: V4L2 Camera HALv3
top: false
cover: false
toc: true
mathjax: true
date: 2020-10-25 14:13:21
tags: 原创
categories:
    - android
    - camera hal
    - usbcamera
summary: 
---

`camera.v4l2` 实现了 **Camera HALv3** 接口，底层调用了 `Video For Linux 2(V4L2)`，因此适用范围比较广泛，可以适配兼容 `V4L2` 接口的所有设备。不过相对于 `Android Camera` 来说，`V4L2` 存在一些局限性，造成一些功能缺失是正常的。
<!--more-->

## 当前状态

`camera.v4l2` 可以自由使用，但是它不是由 **Android Camera team** 官方维护的， 实际上官方维护的是另外一个随着 **Android P** 一起更新发布的实现：[External USB Cameras](https://source.android.com/devices/camera/external-usb-cameras)

该 HAL 实现在现有 Android 系统默认是关闭的，不会编译到系统里面。可以按照如下方式打开

## 编译方法

修改 <device>.mk 文件，增加如下配置：
```Android.mk
USE_CAMERA_V4L2_HAL := true
PRODUCT_PACKAGES += camera.v4l2
PRODUCT_PROPERTY_OVERRIDES += ro.hardware.camera=v4l2
```
第一行会打开编译选项，默认关闭，防止一些设备不支持出现问题。第二行告诉编译系统将库打包到 `system image` 里面；最后一行告诉硬件设备管理器加载 `V4L2 HAL` 替换掉默认的 `Camera HAL`.

## 前提条件
- camera 必须支持 `BGR32`, `YUV420` 和 `JPEG` 格式
- 设备上的 **gralloc** 和其它 **graphics module** 必须使用 `HAL_PIXEL_FORMAT_RGBA_8888` 作为 `HAL_PIXEL_FORMAT_IMPLEMENTATION_DEFINED`

## HAL 代码解析

**V4L2 Camera HAL** 包含 3 个部分：HALv3 Camera 实现， V4L2 wrapper，以及 Metadata

同时，你应该要了解一下 **Android Framework** 是如何与 **HAL** 打交道的，可以参考 [Android Camera HAL 介绍](https://www.xbwee.space/2020/10/23/Android-Camera-HAL-Intro/)

### Camera & HAL 接口

接口实现主要是这两个类：`Camera`, `V4L2CameraHal`

`V4L2CameraHAL` 主要负责相机系统初始化，创建时，会搜索 `dev/video*` 设备节点，并查询是否满足 `V4L2_CAP_VIDEO_CAPTURE` 然后创建对应的 `V4L2Camera` 对象，并且对 Framework 可见。后续流程调用会被 dispatch 到特定的设备节点。

`Camera` 类实现了 camera 设备的通用操作，打开/关闭设备、configuring streams、preparing and tradking requests 等等。具体的拍照、设置流程则是由其子类 `V4L2Camera` 实现。

`Camera` 类在调用具体流程的时候，会应用上 `V4L2Camera` 初始化之后的 `Metadata` 属性，比如 **in-flight** request per stream 数量的限制。换句话说，`Camera` 类实现具体的 HAL 调用流程，与而具体的设备无关，而 `V4L2Camera` 负责设备相关的属性设置、功能实现（透过 `V4L2Wrapper` 类）。所以理论上，可以把 V4L2 实现换成其它某种设备实现，只要传递正确的 metadata 信息，`Camera` 类依然会按照预期的工作。

### V4L2 具体实现

`V4L2Camera` 实现所有拍照功能。它包含一些方法用来获取或设置参数，但它的核心能力主要是在 `request queue` 上。 `Camera` 提交 `CaptureRequest` 到 `request queue` 排队，`V4L2Camera` 异步地从队列里取出来执行，处理过程主要分三个阶段：
- 接收 request 请求：收到 request 并放入等待队列。
- enqueue：读取 request 配置并应用到 v4l2 设备，执行拍照，并传递 buffer 给 v4l2 驱动。
- dequeue：从驱动获取到处理完成的 frame，buffer 内容 copy 到 request output 并传递给 `Camera` 做后续处理（验证结果，填入 `CaptureResult` 并返回给 Framework）

这项工作的大部分是 `V4L2Wrapper` 类完成的，它基于 HAL 的功能需求，包装了 v4l2 ioctls, 提供简单的输入输出调用功能。自动填入 ioctls 需要的常量参数，抽取 HAL 所需信息，同时把相关功能暴露给 Metadata 系统，获取、设置 meta 控制参数。


### Metadata

`Metadata` 子系统主要目的是简化 (system/media/camera/docs/docs.html) 的相关操作。顶层是 `Metadata` 和 `PartialMetadataInterface` 类，`Metadata` 类提供高层次功能，包括初始化 `static metadata`，验证，获取、设定相关配置参数等等，它把这些需求下发给对应的 `PartialMetadataInterfaces` 模块，各个子模块负责处理自己相关的 metadata 和任务。

已经实现了几个具体类负责这项功能，主要有 3 类：
- Properties：静态属性, static metadata 等
- Controls：动态属性，或者标示允许值的静态属性
- States：动态的只读的属性

针对不同的功能需求和不同的 `metadata tags`, 还有一些更具体的接口和子类型来区分处理。

#### Metadata Factory

为了满足 HAL 规范的要求, V4L2 Camera HAL 使用一个 metadata factory 工厂方法，初始化 100+ metadata 参数，大多数参数都是固定值，只有少部分对应到 v4l2 驱动相关的参数。

这套 HAL 实现最初是为了提供给 **Raspberry Pi** camera module v2.1 使用的，所以大多固定默认值的设置主要是适配它的相机模组。


## V4L2 不足之处
- 不支持多个 `stream`，如果 **preview** 与 **capture** 格式不一样，必须重新配置 `stream`; 因此不支持 **Android Camera (v1) API**，只能使用 **Camera2 API**
- 有些 metadata 信息无法从 V4L2 获取到，比如一些物理属性
- Android 系统实现要求 HAL 必须支持 YUV420， JPEG 以及 Graphics 子系统定义的（implementation defined），但实际上只有少数 camera 完整支持这几种格式（比如树莓派相机），因此 HAL 内部实现了格式转换以扩展其适用范围。
- V4L2 并不能确保参数设置立即生效，所以也没有办法确认当前帧给定的设置是否已生效。因此 `CaptureRequest` 和 `CaptureResult` 所带的参数有可能生效，有可能没有。在使用到这些参数的时候，需要注意，它不一定是准确的。
- 另外，V4L2 的很多功能，并没有包含在 HAL 实现里面(比如与camera功能无关的),所以功能上讲, HAL 只是完整 V4L2 的一个子集。


## 已知问题
- 该库未实现的功能包括：high speed capture, flash torch mode, hotplugging/unplugging
