---
title: V4L2 Camera HALv3
top: false
cover: false
toc: true
mathjax: true
date: 2020-10-25 14:13:21
tags: usbcamera
categories:
    - android
    - camera hal
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


### V4L2 wrapper 实现



### Metadata



## V4L2 不足之处
- 不支持多个 `stream`，如果 **preview** 与 **capture** 格式不一样，必须重新配置 `stream`; 因此不支持 **Android Camera (v1) API**，只能使用 **Camera2 API**
- 有些 metadata 信息无法从 V4L2 获取到，比如一些物理属性
- Android 系统实现要求 HAL 必须支持 YUV420， JPEG 以及 Graphics 子系统定义的（implementation defined），但实际上只有少数 camera 完整支持这几种格式（比如树莓派相机），因此 HAL 内部实现了格式转换以扩展其适用范围。
- V4L2 并不能确保参数设置立即生效，所以也没有办法确认当前帧给定的设置是否已生效。因此 `CaptureRequest` 和 `CaptureResult` 所带的参数有可能生效，有可能没有。在使用到这些参数的时候，需要注意，它不一定是准确的。
- 另外，V4L2 的很多功能，并没有包含在 HAL 实现里面(比如与camera功能无关的),所以功能上讲, HAL 只是完整 V4L2 的一个子集。


## 已知问题
- 该库未实现的功能包括：high speed capture, flash torch mode, hotplugging/unplugging
