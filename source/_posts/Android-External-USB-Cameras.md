---
title: Android 外接 USB 摄像头
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-12 17:42:17
tags: 原创
categories:
    - android
    - camera hal
    - usbcamera
summary: 详述 Android 设备开启外接 USB 摄像头支持的实现
---

To be continued...
<!-- 
如前述文章所述，Google 在 Android P 上提供了对 usb camera 设备的支持，官方叫法是 `External USB Cameras` ，提供了完整 HALv3 实现并接入到 `CameraProviderManager`；可以让任何三方相机应用轻松调用到外接 USB 摄像头功能，而且使用方法跟内置相机几无差别，都是透过 `Android Camera API2` 调用。

遗憾的是该功能默认关闭，并且 OEM 厂商大概率也会去改 AOSP 代码，比如 multi-caemra，SAT 等功能的实现，有可能会对其造成影响。

这篇文章详述如何开启 Android 手机上原生支持 USB 外接摄像头这个功能。因为底层走 V4L2 接口，所以支持 UVC 驱动的视频设备都能支持，包括常见的单反，微单，PC 机用的 usb 摄像头，网络摄像头等等，应用非常广泛。

目前看网络上还没有这方面的相关资料，一些嵌入式设备可能有类似的功能实现 (基于 `Linux` 或 `Qt`)，基于 android 系统开发的也有可能是直接使用这套方案的。当然也可以实现自己的 HAL 模块并接入 Android Camera HAL 子系统，具体可参考这篇文章介绍 {% post_link Android-Camera-HAL-Treble  Android Camera HAL 新架构 %}

> 提示：
> 需要 root 权限的手机，并且可以源码编译
> 如果都没有，退而求其次，备选方案可参考这篇文章 {% post_link Android-USB-Camera-Implementations  Android USB Camera 的实现方案 %}

<!-- more -->

首先确保手机系统支持 USB 主机模式 `android.hardware.usb.host`, 一个简单的办法是连接 usb 外设，然后安装运行**USBEnumerator** 这个 android demo app，看是否能列出连接的设备。
> > **备注**  
`USBEnumerator` 获取方式：  
Android Studio -> Import an Android code sample -> 搜索 USB -> 编译安装

当 Android 设备处于 USB 主机模式时，它会充当 USB 主机，为总线供电并枚举连接的 USB 设备。Android 3.1 及更高版本支持 USB 主机模式。

# 修改配置

## kernel 配置

必须开启 kernel 对 UVC 设备的支持，主要是需要 V4L2 驱动层功能模块。
修改 kernel config 文件，增加如下声明
```kconfig
+CONFIG_USB_VIDEO_CLASS=y
+CONFIG_MEDIA_USB_SUPPORT=y
```

## android 配置

external camera 默认是关闭的，需要修改配置脚本打开，以当前的 `2.4` 版本为例说明。


**第一步**，修改产品对应的 product mk 配置文件，增加如下声明:
- 第一行将 external-service 及其启动脚本加入系统
- 第二行是将新增加的 usb camera 配置文件拷贝到 out/target/product/DEVICE/vendor/etc/ 下面，也可以省略这一步，直接 push 到手机。

`device/your_oem/your_product/product_xxx.mk`
```mk
+PRODUCT_PACKAGES += android.hardware.camera.provider@2.4-external-service

+PRODUCT_COPY_FILES += \
+device/manufacturerX/productY/external_camera_config.xml:$(TARGET_COPY_OUT_VENDOR)/etc/external_camera_config.xml
```

编译如下两个模块：
- `hardware/interfaces/camera/provider/2.4`
- `hardware/interfaces/camera/device/3.4`

然后 push 如下几个更新的库到手机 (`DEVICE` 换成对应的product)：
```bash
# external provider
adb push out/target/product/DEVICE/vendor/lib64/android.hardware.camera.provider@2.4-external.so /vendor/lib64/
adb push out/target/product/DEVICE/vendor/lib/android.hardware.camera.provider@2.4-external.so /vendor/lib
# external camera device
adb push out/target/product/DEVICE/vendor/lib64/camera.device@3.4-external-impl.so /vendor/lib64
adb push out/target/product/DEVICE/vendor/lib/camera.device@3.4-external-impl.so /vendor/lib/
# external service
adb push out/target/product/DEVICE/vendor/bin/hw/android.hardware.camera.provider@2.4-external-service /vendor/bin/hw/
# startup config
adb push out/target/product/DEVICE/vendor/etc/init/android.hardware.camera.provider@2.4-external-service.rc /vendor/etc/init/
```

**第二步**， 在 ODM 或者 vendor manifest xml 里面，增加一个节点 `external/0`, hwservicemanager 会读取该配置信息，如果没有对应的节点信息，则无法注册 external HAL 服务，上层看不到 camera 设备。

source tree 位置: `device/VENDOR/DEVICE/[xxx_]manifest.xml`  
手机上位置：`/vendor/etc/vintf/manifest.xml`
```xml
<hal format="hidl">
   <name>android.hardware.camera.provider</name>
   <transport arch="32+64">passthrough</transport>
   <impl level="generic"></impl>
   <version>2.4</version>
   <interface>
       <name>ICameraProvider</name>
       <instance>legacy/0</instance>
+       <instance>external/0</instance>
   </interface>
</hal>
```
直接更新手机上的文件：
```bash
adb pull /vendor/etc/vintf/manifest.xml ./
# 如上修改
adb push manifest.xml /vendor/etc/vintf/
```

**第三步**，增加设备对应的 `external_camera_config.xml` 配置, 手动编辑好之后 push 到手机的 `/vendor/etc/` 下面即可
```
adb push external_camera_config.xml /vendor/etc/
```

`/vendor/etc/external_camera_config.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<ExternalCamera>
    <Provider>
        <ignore> <!-- Internal video devices to be ignored by external camera HAL -->
            <id>0</id> <!-- No leading/trailing spaces -->
            <id>1</id>
            <id>32</id>
            <id>33</id>
        </ignore>
    </Provider>
    <!-- See ExternalCameraUtils.cpp for default values of Device configurations below -->
    <Device>
        <!-- Max JPEG buffer size in bytes-->
        <MaxJpegBufferSize bytes="3145728"/> <!-- 3MB (~= 1080p YUV420) -->
        <!-- Size of v4l2 buffer queue when streaming >= 30fps -->
        <!-- Larger value: more request can be cached pipeline (less janky)  -->
        <!-- Smaller value: use less memory -->
        <NumVideoBuffers count="4"/>
        <!-- Size of v4l2 buffer queue when streaming < 30fps -->
        <NumStillBuffers count="2"/>

        <!-- List of maximum fps for various output sizes -->
        <!-- Any image size smaller than the size listed in Limit row will report
            fps (as minimum frame duration) up to the fpsBound value. -->
        <FpsList>
            <!-- width/height must be increasing, fpsBound must be decreasing-->
            <Limit width="640" height="480" fpsBound="30.0"/>
            <Limit width="1280" height="720" fpsBound="15.0"/>
            <Limit width="1920" height="1080" fpsBound="10.0"/>
            <!-- image size larger than the last entry will not be supported-->
        </FpsList>
    </Device>
</ExternalCamera>
```


**第四步**，可选（目前不需要），如果要支持 `Treble passthrough mode`，则需要更新 `sepolicy` 配置，以便 `camerserver` 可以访问 `UVC camera`  

`device/VENDOR/DEVICE/sepolicy/xxx/hal_camera.te`
```mk
+# for external camera
+allow cameraserver device:dir r_dir_perms;
+allow cameraserver video_device:dir r_dir_perms;
+allow cameraserver video_device:chr_file rw_file_perms;
```

还有个简单直接的办法，调试时暂时关闭 `sepolicy`
```sh
setenforce 0    # 强制关闭 sepolicy
```

# framework 修复

上述动态库和配置文件都 ready 之后，极大可能还会遇到 binder crash，JE 等各种崩溃，或者 App 看不到0，1 之外的其它摄像头设备，这是因为 OEM 厂商动了 AOSP 代码。不同的 OEM 修改不一样。

基本上都是改几行代码就能解决，主要是 `CameraManager.java` 这个文件! 

# 使用外接 usb camera

## Camera2Basic 预览

编译安装 Camera2Basic 这个android demo app，手机连接 usb camera，打开app 界面，会看到有 **“External JPEG(/dev/videox)"** 这个设备，点击打开预览。

{% img /images/Screenshot_2020-11-17-19-19-31-610_Camera2Basic.jpg 270 560 "相机列表" %}

如下是连接单反长焦镜头手机截图，从此单反的大光圈虚化，N 倍光学变焦手机也可以拥有！

{% img /images/Screenshot_2020-11-26-18-42-17-619_Camera2Basic.jpg 270 560 "单反光学变焦" %}

{% img /images/Screenshot_2020-11-26-18-42-40-228_Camera2Basic.jpg 270 560 "单反背景虚化" %}



> **备注**  
`Camera2Basic` 获取方式：  
Android Studio -> Import an Android code sample -> Camera2Basic -> 编译安装


## 微信视频通话

微信默认使用前置摄像头（cameraId=1），只需要让 external usb camera 返回给 CameraProviderManager 看到的 `cameraId` 是 "1" 即可。


> 提示：  
- external provider 调用 `cameraDeviceStatusChanged` 时返回 `cameraId="1"`，确保 CameraProviderManager -> cameraservice -> App 看到的设备id 是 "1" ，替换掉前置。
- external device 强制写死，比如 `mCameraId("/dev/video2")`, 确保 open/close 时能正确打开 V4L2 设备节点
- CameraProviderManager 需要修改，开机时已经检测并添加了 id=1 的前置摄像头设备，因此它不会允许再添加 external usb 设备，这个逻辑需要修改一下。


Enjoy yourself！！！ -->



# 参考资料
> `Device Manifest`   
https://source.android.com/devices/architecture/vintf/objects  
`External USB Cameras`  
https://source.android.com/devices/camera/external-usb-cameras  
`USB Host`  
https://developer.android.com/guide/topics/connectivity/usb/host