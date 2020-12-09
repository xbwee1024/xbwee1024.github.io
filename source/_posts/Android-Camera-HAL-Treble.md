---
title: Android Camera HAL 新架构
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-18 17:40:24
tags: 原创
categories:
    - android
    - camera hal
summary: Treble 架构下的 Android Camera 介绍
---

我们也把 Camera 拆分成 **三驾马车** 来看：`App Framework`, `CameraService`, `Camera HAL`

前面两个是 `Android AOSP` 代码，随着 Android 系统升级会持续更新，包含在 system.img 里面，同时 Java 部分接口也会包含在 `Android SDK` 一起发布。

随着 `Android 8.0` 的 Treble 架构发布，相当于是把三驾马车里面的前两架放到了 `system.img` 里面，一起随着 Android 系统升级更新，而最后一个则独立拆分出来，不仅仅是放到了新的进程里面，而且在手机系统上要求放到 `vendor` 分区，这样可以各自独立升级。

<!-- more -->

# 三驾马车

先附上 Google 官方图例：
{% img /images/ape_fwk_camera2.jpg  374 298 "Camera HAL 新架构" %}

`App Framework` 部分是最上层部分，包括 Java & C++ 代码，实现了 `Android Camera2 API` 接口，提供给 android 应用使用，Java 部分包含在 Android SDK 里面。

> source tree  
- Java 实现：  
    - https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/hardware/
    - https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/hardware/camera2
- C++ 实现：  
    - https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/jni/  


`CameraService` 是中间桥梁，负责沟通 Framework 与 Camera 硬件设备，把上层的调用需求透过 camera hal 接口转发给 HAL 硬件实现，同时返回处理结果。
> source tree  
- C++ 实现：  
    - https://android.googlesource.com/platform/frameworks/av/+/refs/heads/master/camera/  
    - https://android.googlesource.com/platform/frameworks/av/+/refs/heads/master/services/camera/libcameraservice/
    - https://android.googlesource.com/platform/frameworks/hardware/interfaces/+/refs/heads/master/cameraservice/


`Camera HAL` 是硬件适配层，针对不同 camera 硬件模组，由 OEM 厂商提供具体实现
> source tree
- Treble 架构  
    - https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/master/camera/
- Legacy  
    - https://android.googlesource.com/platform/hardware/libhardware/+/refs/heads/master/modules/camera/








# 第二架马车：cameraserver


`frameworks/av/camera/cameraserver/main_cameraserver.cpp`
```c++
#define LOG_TAG "cameraserver"
//#define LOG_NDEBUG 0
#include "CameraService.h"
#include <hidl/HidlTransportSupport.h>
using namespace android;
int main(int argc __unused, char** argv __unused)
{
    signal(SIGPIPE, SIG_IGN);
    // Set 5 threads for HIDL calls. Now cameraserver will serve HIDL calls in
    // addition to consuming them from the Camera HAL as well.
    hardware::configureRpcThreadpool(5, /*willjoin*/ false);
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    ALOGI("ServiceManager: %p", sm.get());
    CameraService::instantiate();
    ALOGI("ServiceManager: %p done instantiate", sm.get());
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}

```
`frameworks/av/camera/cameraserver/cameraserver.rc`
```
service cameraserver /system/bin/cameraserver
    class main
    user cameraserver
    group audio camera input drmrpc
    ioprio rt 4
    task_profiles CameraServiceCapacity MaxPerformance
    rlimit rtprio 10 10
```







# 第三驾马车：HAL Impl
## Camera Provider 实现

首先，实现 hal 独立运行的进程，并向 `hwservicemanager` 注册 `ICameraProvider` 服务，对上层提供硬件功能，包括设备状态信息的查询/更新，获取设备接口以便可以调用设备功能。

AOSP 代码目前有两个 `ICameraProvider` 服务进程：
- `legacy/0`, 给内置相机用, 所以目前为止，Camera HALv3 实现还是属于`传统 HAL`，参考[HAL 类型](https://source.android.google.cn/devices/architecture/hal-types?hl=zh-cn)
- `external/0`, Android P 新增的调用外接 usb 相机的 HAL 服务进程

它们都有对应的启动脚本，在系统启动时加载运行。所以**内置相机**和**外接相机**的HAL 实现是分别在两个不同进程里面，各自独立互不影响！

如果需要还可以扩展更多 camera provider，比如 `internal`, `legacy`, `external`, `remote` 等等，只要具备以下几个条件，`cameraservice` 就能透过 `CameraProviderManager` 查询到该服务并查询到 camera 设备列表，供 `App Framework` 使用该相机设备：
- 启动脚本
- 启动 binary，调用 `defaultPassthroughServiceImplementation` 注册服务
- manifest camera.provider 节点声明，包括 `ßtransport` 类型和 `instance` 实例节点名称

进程启动时，会完成这么几件事，跟 `hwservicemanager`以及`PassthroughServiceManager`的交互通过 binder IPC 调用完成
1. 确保 `/dev/vndbinder` 对应的 binder 服务进程已经启动，如果没有则启动它
2. 在该进程里透过 `android_dlopen_ext/dlopen` 加载对应版本的库，这里是`android.hardware.camera.provider@2.4-impl.so`
3. 透过 `dlsym` 获取到 `HIDL_FETCH_ICameraProvider` 方法，然后遍历 manifest 里面 `camera.provider` 这个 `hidl` 接口声明的 `instance` 实例名称，作为参数调用该方法， 即可获取到该实例对应的 `ICameraProvider` 实现
4. 向 hwservicemanager 注册该服务：
   - 首先在 `hwservicemanager` 中插入一个 `HidlService` (interfaceName, instanceName)
   - 其次生成的 `CameraProvider` 对象也要透过 IPC 调用 `hidl::manager::add` 把自己添加到 `hwservicemanager`，因为上一步已经插入了一个对应的 `HidlService` 所以仅仅是更新 `pid`和 `service` 指向实例对象
   - (interfaceName， instanceName) 对应 internal camera provider 就是 ("android.hardware.camera.provider@2.4::ICameraProvider", "/legacy/0")，

这几个步骤都在当前启动的进程里面调用 `defaultPassthroughServiceImplementation` 完成的，所以调用过程中透过 `IPCThreadState::self()->getCallingPid()` 获取到当前进程的pid，也就是提供服务的进程。

详细代码可以跟踪该函数查阅，另外还要参考 `hardware/interfaces/camera` 源代码下面 `*.hal` 编译时由 `hidl-gen` 生成的中间源代码
{% img /images/ICameraProvider-hal-gen.jpg 364 419 "hidl-gen intermediates" %}


`HidlService` 保持的信息如下，包括**接口名称**, **实例名称**, **实例对象**， **HAL进程pid**
```c++
struct HidlService {
    HidlService(const std::string &interfaceName,
                const std::string &instanceName,
                const sp<IBase> &service,
                const pid_t pid);
    HidlService(const std::string &interfaceName,
                const std::string &instanceName)
    : HidlService(
        interfaceName,
        instanceName,
        nullptr,
        static_cast<pid_t>(IServiceManager::PidConstant::NO_PID))
    {}
    virtual ~HidlService() {}
    ...
```

这里可以看到，同一个 hidl 接口，可以有多个实例实现，对应不同类型的硬件设备，比如这里 `/legacy/0` 对应内置相机，`/external/0` 对应外接 usb 相机，它们的 `interfaceName` 一样，`instanceName` 不一样。并且各自有自己的独立进程。


`camera.provider` 接口声明，包括 transport 是 `hwbinder`, 实例有两个 `legacy/0`, `external/0`

`manifest.xml`
```xml
    <hal format="hidl">
        <name>android.hardware.camera.provider</name>
        <transport>hwbinder</transport>
        <version>2.4</version>
        <interface>
            <name>ICameraProvider</name>
            <instance>legacy/0</instance>
            <instance>external/0</instance>
        </interface>
        <fqname>@2.4::ICameraProvider/legacy/0</fqname>
    </hal>
```


`hardware/interfaces/camera/provider/2.4/default/service.cpp`
```c++
int main()
{
    ALOGI("CameraProvider@2.4 legacy service is starting.");
    // The camera HAL may communicate to other vendor components via
    // /dev/vndbinder
    android::ProcessState::initWithDriver("/dev/vndbinder");
    status_t status;
    if (kLazyService) {
        status = defaultLazyPassthroughServiceImplementation<ICameraProvider>("legacy/0",
                                                                              /*maxThreads*/ 6);
    } else {
        status = defaultPassthroughServiceImplementation<ICameraProvider>("legacy/0",
                                                                          /*maxThreads*/ 6);
    }
    return status;
}
```


`hardware/interfaces/camera/provider/2.4/default/external-service.cpp`
```c++
int main()
{
    ALOGI("External camera provider service is starting.");
    // The camera HAL may communicate to other vendor components via
    // /dev/vndbinder
    android::ProcessState::initWithDriver("/dev/vndbinder");
    return defaultPassthroughServiceImplementation<ICameraProvider>("external/0", /*maxThreads*/ 6);
}
```

注意下面 `extern "C"` 的声明，确保 C++ 符合没有做 **name mangling** 处理，保持原样，确保服务进程启动时可以透过 `dlsym` 获取到 **HIDL_FETCH_Interface** 并构造服务实例对象，然后向`hwservicemanager`注册该服务。

同时 `CameraProvider` 类的实现是一个模板类工厂，根据不同模板参数创建对应的类实例。


`hardware/interfaces/camera/provider/2.4/default/CameraProvider_2_4.cpp`
```c++
#include "CameraProvider_2_4.h"
#include "LegacyCameraProviderImpl_2_4.h"
#include "ExternalCameraProviderImpl_2_4.h"

const char *kLegacyProviderName = "legacy/0";
const char *kExternalProviderName = "external/0";

namespace android {
namespace hardware {
namespace camera {
namespace provider {
namespace V2_4 {
namespace implementation {

using android::hardware::camera::provider::V2_4::ICameraProvider;

extern "C" ICameraProvider* HIDL_FETCH_ICameraProvider(const char* name);

template<typename IMPL>
CameraProvider<IMPL>* getProviderImpl() {
    CameraProvider<IMPL> *provider = new CameraProvider<IMPL>();
    if (provider == nullptr) {
        ALOGE("%s: cannot allocate camera provider!", __FUNCTION__);
        return nullptr;
    }
    if (provider->isInitFailed()) {
        ALOGE("%s: camera provider init failed!", __FUNCTION__);
        delete provider;
        return nullptr;
    }
    return provider;
}

ICameraProvider* HIDL_FETCH_ICameraProvider(const char* name) {
    using namespace android::hardware::camera::provider::V2_4::implementation;
    ICameraProvider* provider = nullptr;
    if (strcmp(name, kLegacyProviderName) == 0) {
        provider = getProviderImpl<LegacyCameraProviderImpl_2_4>();
    } else if (strcmp(name, kExternalProviderName) == 0) {
        provider = getProviderImpl<ExternalCameraProviderImpl_2_4>();
    } else {
        ALOGE("%s: unknown instance name: %s", __FUNCTION__, name);
    }

    return provider;
}

}  // namespace implementation
}  // namespace V2_4
}  // namespace provider
}  // namespace camera
}  // namespace hardware
}  // namespace android
```

{% img /images/android_camera_hal.svg 514 953 %}


# 自研 HAL 接入的方式

{% img /images/hal_ext.svg 897 793 %}


To be continued...





