---
title: Android USB Camera 的支持与实现
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-02 14:36:18
tags: 原创
categories:
    - android
    - camera hal
    - usbcamera
summary: Summary of current implementations to use usb camera device on Android phone
---

Android 设备基于 `linux kernel`, 自带 `V4L2` 支持，但是 OEM 厂商实现不同，大多默认关闭该功能。所以一般开发者或终端用户想要在 Android 设备上使用 usb camera 不是一件容易的事情。

这里简单介绍几种针对开发者来说，可选择的实现方案.

<!-- more -->


# 基于 libuvc 开发

[libuvc](https://github.com/libuvc/libuvc) 是一个跨平台开发库，基于 `libusb`，功能包括 UVC 设备识别与控制，视频流传输，视频流格式转换等。

Android 平台上已有一个 Usb Camera 的开源项目，基于 `libucv` 的Android 应用，[UVCCamera](https://github.com/saki4510t/UVCCamera) 无需 `root` 权限即可预览显示连接到手机的 usb camera 设备。


`libuvc` 官网介绍：
> libuvc is a cross-platform library for USB video devices, built atop libusb. It enables fine-grained control over USB video devices exporting the standard USB Video Class (UVC) interface, enabling developers to write drivers for previously unsupported devices, or just access UVC devices in a generic fashion.

> libuvc is a library that supports enumeration, control and streaming for USB Video Class (UVC) devices, such as consumer webcams.
Features
- UVC device discovery and management API
- Video streaming (device to host) with asynchronous/callback and synchronous/polling modes
- Read/write access to standard device settings
- Conversion between various formats: RGB, YUV, JPEG, etc.
- Tested on Mac and Linux, portable to Windows and some BSDs

代码支持 CMake，android 平台编译：
```bash
git clone https://github.com/libuvc/libuvc
cd libuvc
mkdir build
cd build
cmake \
    -D CMAKE_INSTALL_PREFIX=your_install_path \
    -D CMAKE_TOOLCHAIN_FILE=${ANDROID_NDK}/build/cmake/android.toolchain.cmake \
    -D CMAKE_BUILD_TYPE=Debug \
    -D ANDROID_NDK=${ANDROIDz_NDK} \
    -D ANDROID_PLATFORM=android-28 \
    -D ANDROID_STL=c++_static \
    -D ANDROID_PIE=ON \
    ..
cmake --build . --config Debug --target install -- -j6
```

UVCCamera
> library and sample to access to UVC web camera on non-rooted Android device   
Copyright (c) 2014-2017 saki t_saki@serenegiant.com



# Android 官方推出的 ExternalCamera

随着 Android P 版本升级，新增了 `External USB Cameras` 这个功能，默认情况该功能是关闭的，一些 HAL 组件不会编译到 ROM 中，需要打开更新 ROM 才行。另外该功能还依赖于 `android.hardware.usb.host` 以及 Linux kernel 打开 `UVC` 驱动支持。

该实现 HAL 会启动一个 `hotplug` 线程，监视 `/dev/video*` 设备节点增删情况，透过 HAL 回调函数通知 `CameraProviderManager` 更新 camera 设备列表。


详细实操过程可以参考这篇文章：
{% post_link Android-External-USB-Cameras Android External USB Cameras %}


# camera.v4l2 实现

该库也是基于 `V4L2` 的 `Camera HALv3` 实现，原本是 Google 开发出来给树莓派系统使用的。所以从 Android AOSP 代码库里面可以找到这份源代码，但是只有 HAL 实现，没有接入 Android Framework，也就是 cameraserver 是调用不到的。如果有树莓派源代码的话倒是可以参考看看，不过估计也是基于这个初阶版本改过甚至是采用了全新的实现。

该库在 Android 系统里也是默认关闭的，需要打开才会编到 ROM 里，代码实现上解耦了 camera interface 与 V2L2 wrapper 部分，所以理论上可以把 V4L2 实现替换成其它也是 ok 的。


详细介绍可以参考这篇文章：
{% post_link V4L2-Camera-HALv3 V4L2 Camera HALv3 介绍 %}



# usbcamera HAL
Google 早期提供的一个示例代码，是空实现。略过。  

代码位置：
[`hardware/libhardware/modules/usbcamera`](https://android.googlesource.com/platform/hardware/libhardware/+/refs/heads/master/modules/usbcamera/)



# 参考资料
> `libuvc`   
https://github.com/saki4510t/UVCCamera   
https://github.com/libuvc/libuvc   
https://github.com/libusb/libusb   
https://ken.tossell.net/libuvc/doc/   
https://libusb.info/

> `external usb camera`  
https://source.android.com/devices/camera/external-usb-cameras    
https://groups.google.com/g/android-platform/c/Qx1P0I17uzs?pli=1

> `V4L2 Camera Hal`   
https://android.googlesource.com/platform/hardware/libhardware/+/master/modules/camera/3_4/README.md
