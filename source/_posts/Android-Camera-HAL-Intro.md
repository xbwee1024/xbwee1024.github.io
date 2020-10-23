---
title: Android Camera HAL 介绍
top: false
cover: false
toc: true
mathjax: true
date: 2020-10-23 10:48:12
tags:
categories: camera hal
summary: android camera hal 介绍
---

Android Camera HAL 截止目前最新版本是 HALv3，高端机型基本上都支持。熟悉了 [Android HAL](Android-HAL-Intro) 接口继承关系，模块搜索、加载，设备打开、关闭的通用流程，再来看 Camera HAL 会更容易理解。

不过 Camera HAL 属于相对比较复杂的一个硬件模块，随着硬件模组的发展，HAL 接口也在不停演进。模块接口增加了几个通用接口，设备接口层面则演化到最新的 HALv3。 v1, v2 对应 `android.hardware.Camera` API, 已经不再被支持。推荐实现 v3.2 及以上。

模块名称定义如下，加载模块时根据模块名称搜索对应的动态库 `camera.variant.so`。
```c++
#define CAMERA_HARDWARE_MODULE_ID "camera"
```

<!--more-->

```c++
>>> libhardware/include/hardware/camera_common.h
typedef struct camera_module {
    /**
     * Common methods of the camera module.  This *must* be the first member of
     * camera_module as users of this structure will cast a hw_module_t to
     * camera_module pointer in contexts where it's known the hw_module_t
     * references a camera_module.
     */
    hw_module_t common;

    int (*get_number_of_cameras)(void);

    int (*get_camera_info)(int camera_id, struct camera_info *info);

    /**
     * Version information (based on camera_module_t.common.module_api_version):
     *
     *  CAMERA_MODULE_API_VERSION_1_0, CAMERA_MODULE_API_VERSION_2_0:
     *
     *    Not provided by HAL module. Framework may not call this function.
     *
     *  CAMERA_MODULE_API_VERSION_2_1:
     *
     *    Valid to be called by the framework.
     */
    int (*set_callbacks)(const camera_module_callbacks_t *callbacks);

    /**
     * Version information (based on camera_module_t.common.module_api_version):
     *
     *  CAMERA_MODULE_API_VERSION_1_x/2_0/2_1:
     *    Not provided by HAL module. Framework may not call this function.
     *
     *  CAMERA_MODULE_API_VERSION_2_2:
     *    Valid to be called by the framework.
     */
    void (*get_vendor_tag_ops)(vendor_tag_ops_t* ops);

    /**
     * Version information (based on camera_module_t.common.module_api_version):
     *
     *  CAMERA_MODULE_API_VERSION_1_x/2_0/2_1/2_2:
     *    Not provided by HAL module. Framework will not call this function.
     *
     *  CAMERA_MODULE_API_VERSION_2_3:
     *    Valid to be called by the framework.
     */
    int (*open_legacy)(const struct hw_module_t* module, const char* id,
            uint32_t halVersion, struct hw_device_t** device);

    /**
     * Version information (based on camera_module_t.common.module_api_version):
     *
     * CAMERA_MODULE_API_VERSION_1_x/2_0/2_1/2_2/2_3:
     *   Not provided by HAL module. Framework will not call this function.
     *
     * CAMERA_MODULE_API_VERSION_2_4:
     *   Valid to be called by the framework.
     */
    int (*set_torch_mode)(const char* camera_id, bool enabled);

    /**
     * Version information (based on camera_module_t.common.module_api_version):
     *
     * CAMERA_MODULE_API_VERSION_1_x/2_0/2_1/2_2/2_3:
     *   Not provided by HAL module. Framework will not call this function.
     *
     * CAMERA_MODULE_API_VERSION_2_4:
     *   If not NULL, will always be called by the framework once after the HAL
     *   module is loaded, before any other HAL module method is called.
     */
    int (*init)();
    
    /**
     * Version information (based on camera_module_t.common.module_api_version):
     * 
     * CAMERA_MODULE_API_VERSION_1_x/2_0/2_1/2_2/2_3/2_4:
     *   Not provided by HAL module. Framework will not call this function.
     *
     * CAMERA_MODULE_API_VERSION_2_5 or higher:
     *   If any of the camera devices accessible through this camera module is
     *   a logical multi-camera, and at least one of the physical cameras isn't
     *   a stand-alone camera device, this function will be called by the camera
     *   framework. Calling this function with invalid physical_camera_id will
     *   get -EINVAL, and NULL static_metadata.
     */
    int (*get_physical_camera_info)(int physical_camera_id,
            camera_metadata_t **static_metadata);

    /**
     * Version information (based on camera_module_t.common.module_api_version):
     * 
     * CAMERA_MODULE_API_VERSION_1_x/2_0/2_1/2_2/2_3/2_4:
     *   Not provided by HAL module. Framework will not call this function.
     *
     * CAMERA_MODULE_API_VERSION_2_5 or higher:
     *   Valid to be called by the framework.
     */
    int (*is_stream_combination_supported)(int camera_id,
            const camera_stream_combination_t *streams);

    /**
     * notify_device_state_change:
     *
     * Notify the camera module that the state of the overall device has
     * changed in some way that the HAL may want to know about.
     */
    void (*notify_device_state_change)(uint64_t deviceState);

    /**
     * get_camera_device_version:
     *
     * Return the device version for a given camera device. This value may not change for a camera
     * device. The version returned here must be the same as the one from get_camera_info.
     */
    int (*get_camera_device_version)(int camera_id, uint32_t *version);

    /* reserved for future use */
    void* reserved[1];
} camera_module_t;
```

```c++
>>> libhardware/include/hardware/camera3.h
typedef struct camera3_device {
    /**
     * common.version must equal CAMERA_DEVICE_API_VERSION_3_0 to identify this
     * device as implementing version 3.0 of the camera device HAL.
     *
     * Performance requirements:
     *
     * Camera open (common.module->common.methods->open) should return in 200ms, and must return
     * in 500ms.
     * Camera close (common.close) should return in 200ms, and must return in 500ms.
     *
     */
    hw_device_t common;
    camera3_device_ops_t *ops;
    void *priv;
} camera3_device_t;

>>> libhardware/include/hardware/camera2.h
typedef struct camera2_device {
    /**
     * common.version must equal CAMERA_DEVICE_API_VERSION_2_0 to identify
     * this device as implementing version 2.0 of the camera device HAL.
     */
    hw_device_t common;
    camera2_device_ops_t *ops;
    void *priv;
} camera2_device_t;

>>> libhardware/include/hardware/camera.h
typedef struct camera_device {
    /**
     * camera_device.common.version must be in the range
     * HARDWARE_DEVICE_API_VERSION(0,0)-(1,FF). CAMERA_DEVICE_API_VERSION_1_0 is
     * recommended.
     */
    hw_device_t common;
    camera_device_ops_t *ops;
    void *priv;
} camera_device_t;
```

```c++
>>> libhardware/include/hardware/camera3.h

typedef struct camera3_device_ops {

    /**
     * initialize:
     *
     * One-time initialization to pass framework callback function pointers to
     * the HAL. Will be called once after a successful open() call, before any
     * other functions are called on the camera3_device_ops structure.
     *
     * Performance requirements:
     *
     * This should be a non-blocking call. The HAL should return from this call
     * in 5ms, and must return from this call in 10ms.
     */
    int (*initialize)(const struct camera3_device *,
            const camera3_callback_ops_t *callback_ops);

    /**
     * Performance requirements:
     *
     * The HAL should return from this call in 500ms, and must return from this
     * call in 1000ms.
     */
    int (*configure_streams)(const struct camera3_device *,
            camera3_stream_configuration_t *stream_list);

    /**
     * Performance requirements:
     *
     * This should be a non-blocking call. The HAL should return from this call
     * in 1ms, and must return from this call in 5ms.
     */
    int (*register_stream_buffers)(const struct camera3_device *,
            const camera3_stream_buffer_set_t *buffer_set);

    /**
     * Performance requirements:
     *
     * This should be a non-blocking call. The HAL should return from this call
     * in 1ms, and must return from this call in 5ms.
     *
     */
    const camera_metadata_t* (*construct_default_request_settings)(
            const struct camera3_device *,
            int type);

    /**
     * Performance considerations:
     *
     * Handling a new buffer should be extremely lightweight and there should be
     * no frame rate degradation or frame jitter introduced.
     *
     * This call must return fast enough to ensure that the requested frame
     * rate can be sustained, especially for streaming cases (post-processing
     * quality settings set to FAST). The HAL should return this call in 1
     * frame interval, and must return from this call in 4 frame intervals.
     */
    int (*process_capture_request)(const struct camera3_device *,
            camera3_capture_request_t *request);

    /**
     * >= CAMERA_DEVICE_API_VERSION_3_2:
     *    DEPRECATED. This function has been deprecated and should be set to
     *    NULL by the HAL.  Please implement get_vendor_tag_ops in camera_common.h
     *    instead.
     */
    void (*get_metadata_vendor_tag_ops)(const struct camera3_device*,
            vendor_tag_query_ops_t* ops);

    /**
     * dump:
     *
     * Print out debugging state for the camera device. This will be called by
     * the framework when the camera service is asked for a debug dump, which
     * happens when using the dumpsys tool, or when capturing a bugreport.
     *
     * The passed-in file descriptor can be used to write debugging text using
     * dprintf() or write(). The text should be in ASCII encoding only.
     *
     * Performance requirements:
     *
     * This must be a non-blocking call. The HAL should return from this call
     * in 1ms, must return from this call in 10ms. This call must avoid
     * deadlocks, as it may be called at any point during camera operation.
     * Any synchronization primitives used (such as mutex locks or semaphores)
     * should be acquired with a timeout.
     */
    void (*dump)(const struct camera3_device *, int fd);

    /**
     * Performance requirements:
     *
     * The HAL should return from this call in 100ms, and must return from this
     * call in 1000ms. And this call must not be blocked longer than pipeline
     * latency (see S7 for definition).
     *
     * Version information:
     *
     *   only available if device version >= CAMERA_DEVICE_API_VERSION_3_1.
     *
     */
    int (*flush)(const struct camera3_device *);

    /**
     * signal_stream_flush:
     *
     * <= CAMERA_DEVICE_API_VERISON_3_5:
     *
     *    Not defined and must be NULL
     *
     * >= CAMERA_DEVICE_API_VERISON_3_6:
     *
     */
    void (*signal_stream_flush)(const struct camera3_device*,
            uint32_t num_streams,
            const camera3_stream_t* const* streams);

    /**
     * is_reconfiguration_required:
     *
     * <= CAMERA_DEVICE_API_VERISON_3_5:
     *
     *    Not defined and must be NULL
     *
     * >= CAMERA_DEVICE_API_VERISON_3_6:
     */
    int (*is_reconfiguration_required)(const struct camera3_device*,
            const camera_metadata_t* old_session_params,
            const camera_metadata_t* new_session_params);

    /* reserved for future use */
    void *reserved[6];
} camera3_device_ops_t;
```


## module API version

```c++
>>> include/hardware/camera_common.h
/**
 * All module versions <= HARDWARE_MODULE_API_VERSION(1, 0xFF) must be treated
 * as CAMERA_MODULE_API_VERSION_1_0
 */
#define CAMERA_MODULE_API_VERSION_1_0 HARDWARE_MODULE_API_VERSION(1, 0)
#define CAMERA_MODULE_API_VERSION_2_0 HARDWARE_MODULE_API_VERSION(2, 0)
#define CAMERA_MODULE_API_VERSION_2_1 HARDWARE_MODULE_API_VERSION(2, 1)
#define CAMERA_MODULE_API_VERSION_2_2 HARDWARE_MODULE_API_VERSION(2, 2)
#define CAMERA_MODULE_API_VERSION_2_3 HARDWARE_MODULE_API_VERSION(2, 3)
#define CAMERA_MODULE_API_VERSION_2_4 HARDWARE_MODULE_API_VERSION(2, 4)
#define CAMERA_MODULE_API_VERSION_2_5 HARDWARE_MODULE_API_VERSION(2, 5)

#define CAMERA_MODULE_API_VERSION_CURRENT CAMERA_MODULE_API_VERSION_2_5
```

```c++
camera_module_t.common.module_api_version
```
- major: MSB 16bit
- minor: LSB 16bit

### Versions: 0.X - 1.X [CAMERA_MODULE_API_VERSION_1_0]
- HAL interface 的初版 module 实现
- 透过该版本 module 可打开的 camera 设备，仅仅支持 android.hardware.Camera API 
- `camera_info.device_version` 无效
- `camera_info.static_camera_characteristics` 无效

### Version: 2.0 [CAMERA_MODULE_API_VERSION_2_0]
- 支持 1.0， 2.0 HAL 接口
- `camera_info.device_version` 有效
- `camera_info.static_camera_characteristics` 在 device_version >= 2.0 时有效

### Version: 2.1 [CAMERA_MODULE_API_VERSION_2_1]
- 增加异步回调支持，用来通知 framework 底层状态的变化
- `set_callbacks` 必须是该版本及其以上才支持

### Version: 2.2 [CAMERA_MODULE_API_VERSION_2_2]
- module 层面增加 vendor tag 支持
- `vendor_tag_query_ops` 透过打开的 device 才能调用，不再推荐使用

### Version: 2.3 [CAMERA_MODULE_API_VERSION_2_3]
- 支持打开 legacy device
- 前提是 device 具备多个 device API 实现
- `common.methods->open` 默认打开的还是最新版本的 device

### Version: 2.4 [CAMERA_MODULE_API_VERSION_2_4]
- Torch mode 支持，framework 可以在不打开 device 情况下使用 torch mode；当然 camera device 具有最高优先级使用 flash；当打开相机设备时，HAL 透过 callback 通知 framework torch mode 已关闭
- external camera （例如 USB camera）支持. `CAMERA_DEVICE_STATUS_PRESENT` 才可获取设备状态，否则无效。framework 注册 callback 接收hot-plug camera 状态，并更新可用的设备列表
- Camera arbitration hints， 增加提示信息：同时打开使用的相机个数， `get_camera_info` 调用返回后，`camera_info.resource_cost`, `camera_info.conflicting_devices`必须被设置
- module 初始化支持，在模块被加载后，其它方法被调用之前，初始化被调用，做一次性的初始化工作

### ersion: 2.5 [CAMERA_MODULE_API_VERSION_2_5]
- 可查询对象可以是逻辑相机，而不仅仅是物理相机设备
- 增加 stream combination 能力查询
- 完整的设备状态通知，比如 shutter 的 folding/unfolding, covering/uncovering



## device API version 

```c++
/**
 * All device versions <= HARDWARE_DEVICE_API_VERSION(1, 0xFF) must be treated
 * as CAMERA_DEVICE_API_VERSION_1_0
 */
#define CAMERA_DEVICE_API_VERSION_1_0 HARDWARE_DEVICE_API_VERSION(1, 0) // DEPRECATED
#define CAMERA_DEVICE_API_VERSION_2_0 HARDWARE_DEVICE_API_VERSION(2, 0) // NO LONGER SUPPORTED
#define CAMERA_DEVICE_API_VERSION_2_1 HARDWARE_DEVICE_API_VERSION(2, 1) // NO LONGER SUPPORTED
#define CAMERA_DEVICE_API_VERSION_3_0 HARDWARE_DEVICE_API_VERSION(3, 0) // NO LONGER SUPPORTED
#define CAMERA_DEVICE_API_VERSION_3_1 HARDWARE_DEVICE_API_VERSION(3, 1) // NO LONGER SUPPORTED
#define CAMERA_DEVICE_API_VERSION_3_2 HARDWARE_DEVICE_API_VERSION(3, 2)
#define CAMERA_DEVICE_API_VERSION_3_3 HARDWARE_DEVICE_API_VERSION(3, 3)
#define CAMERA_DEVICE_API_VERSION_3_4 HARDWARE_DEVICE_API_VERSION(3, 4)
#define CAMERA_DEVICE_API_VERSION_3_5 HARDWARE_DEVICE_API_VERSION(3, 5)
#define CAMERA_DEVICE_API_VERSION_3_6 HARDWARE_DEVICE_API_VERSION(3, 6)

// Device version 3.5 is current, older HAL camera device versions are not
// recommended for new devices.
#define CAMERA_DEVICE_API_VERSION_CURRENT CAMERA_DEVICE_API_VERSION_3_5
```

Camera device HAL 3.6[ CAMERA_DEVICE_API_VERSION_3_6 ], 是当前推荐支持的版本
- 支持 `android.hardware.Camera` API
- 有限的 `android.hardware.Camera2` API

CAMERA_DEVICE_API_VERSION_3_2 及以上
- `camera_module_t.common.module_api_version` 最低要求 `2.2`

CAMERA_DEVICE_API_VERSION_3_1 及以下
- `camera_module_t.common.module_api_version` 最低要求 `2.0`

CAMERA_DEVICE_API_VERSION_2_0 及以下
- `camera_module_t.common.module_api_version` 要求`1.0`

### 1.0: Initial Android camera HAL (Android 4.0) [camera.h]
- 由 C++ `CameraHardwareInterface` 抽象接口转换而来
- 支持 `android.hardware.Camera` API

### 2.0: Initial release of expanded-capability HAL (Android 4.2) [camera2.h]
- 完整实现 `android.hardware.Camera` API
- 允许 camera service 持有 ZSL queue
- 新增功能尚未做完整测试：manual capture control, Bayer RAW capture, reprocessing of RAW data

### 3.0: First revision of expanded-capability HAL
- 最低要求 module 版本 `2.0` 及以上
- 重写了 input request 和 stream queue 接口
- 合并 `trigger` 到 `request` 里面；合并`notification`到`results`
- 合并所有`callback`到一个结构体里面；合并初始化设置到一个初始化函数调用里
- 合并`stream`配置到一个调用里；双向流替换`STREAM_FROM_STREAM`结构
- 旧版本的有限支持

### 3.1: Minor revision of expanded-capability HAL
- `configure_streams`传递`consumer usage flags`到 HAL 
- `flush`调用将丢弃所有未处理完的 `requests/buffers`

### 3.2: Minor revision of expanded-capability HAL
- 废弃`get_metadata_vendor_tag_ops`，使用`get_vendor_tag_ops`代替
- 废弃`register_stream_buffers`，`process_capture_request`传递给 HAL 的 gralloc buffer 是由 framework 新分配的
- 新增`partial result`支持，可以多次调用`process_capture_result`，获取到的是调用时刻 ready 的结果，是最终结果的子集
- 新增`manual templete` 到 `camera3_request_template`，应用可直接用来控制拍照设置
- 重写双向流、输入流的定义 spec
- 改变返回结果的路径，由`process_capture_result`返回，不再是`process_capture_request`

### 3.3: Minor revision of expanded-capability HAL
- API 更新：OPAQUE, YUV reprocessing
- 增加深度图输出支持
- `camera3_stream_t`增加`data_space`, `rotation`
- `camera3_stream_configuration_t`增加 camera3 流配置模式

### 3.4: Minor additions to supported metadata and changes to data_space support
- `RAW_OPAQUE`格式增加 `ANDROID_SENSOR_OPAQUE_RAW_SIZE`参数
- Raw 格式增加 `ANDROID_CONTROL_POST_RAW_SENSITIVITY_BOOST_RANGE` 参数
- 重定义`camera3_stream_t.data_space`，用法更灵活
- HALv3.2 以上增加如下几个通用 metadata
    - ANDROID_INFO_SUPPORTED_HARDWARE_LEVEL_3
    - ANDROID_CONTROL_POST_RAW_SENSITIVITY_BOOST
    - ANDROID_CONTROL_POST_RAW_SENSITIVITY_BOOST_RANGE
    - ANDROID_SENSOR_DYNAMIC_BLACK_LEVEL
    - ANDROID_SENSOR_DYNAMIC_WHITE_LEVEL
    - ANDROID_SENSOR_OPAQUE_RAW_SIZE
    - ANDROID_SENSOR_OPTICAL_BLACK_REGIONS

### 3.5: Minor revisions to support session parameters and logical multi camera:
- 增加`ANDROID_REQUEST_AVAILABLE_SESSION_KEYS` metadata 参数
- 增加`camera3_stream_configuration.session_parameters`, 保存`ANDROID_REQUEST_AVAILABLE_SESSION_KEYS`对应的初始化值
- 增加 metadata 支持逻辑相机功能
    - ANDROID_REQUEST_AVAILABLE_CAPABILITIES_LOGICAL_MULTI_CAMERA
    - ANDROID_LOGICAL_MULTI_CAMERA_PHYSICAL_IDS
    - ANDROID_LOGICAL_MULTI_CAMERA_SYNC_TYPE
- 增加`camera3_stream.physical_camera_id`，逻辑相机应用可以单独设定某个物理相机设备的流配置
- 增加`camera3_capture_request.physcam_settings`, 逻辑相机应用可以单独设定某个物理相机设备

### 3.6: Minor revisions to support HAL buffer management APIs:
- 增加`ANDROID_INFO_SUPPORTED_BUFFER_MANAGEMENT_VERSION`，判断是否支持buffer管理
- 增加`camera3_callback_ops_t` 几个 callback， 3.6 以下未定义，必须置空。
    - `request_stream_buffers`, 异步callback，向 camera service 申请 output buffer
    - `return_stream_buffers`, 异步callback，返回给 camera service 填好的 output buffer
    - `signal_stream_flush`, camera service 通知 HAL 即将调用 `configure_streams`, HAL 需要立即完成所有 request 并返回所有 buffers，超时则 camera service 会触发fatal error；如果 HAL 已经返回所有 buffer，则该调用被忽略
    - `is_reconfiguration_required`, 根据新的session 参数，判断是否要重新配置流
- 增加`CAMERA3_JPEG_APP_SEGMENTS_BLOB_ID`



## Startup and operation sequencing

Camera HAL 启动和调用流程，
1. Framework 调用`camera_module_t->common.open()`, 返回`hardware_device_t`结构.
2. Framework 根据`hardware_device_t->version`的值，把返回的对象实例转换成对应版本的 device handler； 比如 `CAMERA_DEVICE_API_VERSION_3_0`则转换成`camera3_device_t`
3. Framework 调用`camera3_device_t->ops->initialize`, 是在`open`之后其它功能函数调用之前
4. Framework 调用`camera3_device_t->ops->configure_streams`, input/output stream参数传递给 HAL
5. 注册输出流
    - `CAMERA_DEVICE_API_VERSION_3_1`及以下版本，Framework 分配好 gralloc buffer，并调用`camera3_device_t->ops->register_stream_buffers`，注册输出流，每个流仅需注册一次
    - `CAMERA_DEVICE_API_VERSION_3_2`及以上版本，`camera3_device_t->ops->register_stream_buffers`置空
6. Framework 调用`camera3_device_t->ops->construct_default_request_settings`，获取特定 usecase 的默认设置， 可以在 step3 之后的任意时刻调用
7. Framework 基于获取的 usecase 默认设置，调用`camera3_device_t->ops->process_capture_request`发起第一个 capture request 到 HAL，至少有一个已注册的输出流，这是一个同步调用，HAL 会block 住直到可以处理下一个 request 才返回
    - `CAMERA_DEVICE_API_VERSION_3_2`及以上，request 带下来的`buffer_handle_t`必须是全新的对象
8. Framework 继续发起 request
    - 需要的话调用`construct_default_request_settings`获取 usecase 默认设置
    - 3.1 及以下，对于尚未注册的输出流，需要调用`register_stream_buffers`注册
9. 拍照动作开始，Sensor 开始曝光
    - 3.2 及以上，HAL 调用`camera3_callback_ops_t->notify` 通知上层 **SHUTTER event**, 包括 `frame number` 和 `timestamp`; 因为 framework 必须要有 timestamp 才能 deliver gralloc buffer 给应用，所以 `notify` 必须尽可能早
    - 3.1 及以下，必须在调用`process_capture_result`之前调用`notify`
    - **partial metadata results** 和 **gralloc buffer** 可以在 **SHUTTER event** 前或者后发送给 Framework

10. pipeline 处理需要时间，pipeline 处理完，HAL 调用`camera3_callback_ops_t->process_capture_result`通知 Framework 返回结果。顺序与发起 request 的顺序一一对应。
    - 3.2 及以上，返回之前，buffer 对应的`release_fence`会被调用，所有权移交给 Framework
    - HAL 可以调用多次`process_capture_result` 更新 **partial metadata results**, Framework 负责合成最终的结果
    - 特别地，对于第 N 帧和第 N+1 帧，该回调可以被同时调到

11. 一段时间之后，Framework 停止发送 request，并等待所有 capture 处理完成。然后又调用 `configure_streams`, 这会重置相机硬件和 pipeline，重新配置输入输出流，流可以被复用，。并且已注册过的输出流不需要再注册。如果至少有一个注册流，则重复 step7 开始的过程。如果没有注册流，则重复 step5 之后的过程。

12. Framework 调用 `camera3_device_t->common->close()` 关闭相机设备，该调用是同步调用过程，需要等待所有输出返回。close 返回之后，HAL 不再允许调用 `camera3_callback_ops_t` 的任何方法

13. 当错误发生,或者有异步事件时, HAL 调用 `camera3_callback_ops_t->notify()` 通知 Framework。当发生 **fatal error** 之后，HAL 应该按照 **close** 类似的处理流程，取消或强制执行完所有任务之后再调用 `notify`, 确保 Framework 在此期间不会收到任何回调。



