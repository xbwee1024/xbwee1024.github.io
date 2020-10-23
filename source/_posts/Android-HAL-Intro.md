---
title: Android HAL 介绍
top: false
cover: false
toc: true
mathjax: true
date: 2020-10-22 10:48:12
tags:
categories: android hal
summary: android hal 介绍，主要是 `hw_module_t`, `hw_device_t` 以及模块加载
---

一个硬件设备对应一个模块（动态库），可以各自独立更新，模块版本与设备版本需要保持兼容性，也就是特定版本的模块，只能加载与之兼容的设备。通常情况模块具有向后兼容性，支持最高版本及其以下版本的设备。

模块封装了该类型设备的通用操作，接口很少发生变动或者只是增加新接口，确保二进制兼容性，兼容低版本设备。

设备封装了特定版本的硬件设备，接口变化更频繁，类似camera 从 HALv1, HALv2 到目前的 HALv3，封装成不同的设备子类，具体硬件根据其硬件特性选择实现不同版本的 device 接口。
<!--more-->

`hw_device_t` 作为 device 的基类，放置于继承的具体设备类型 struct 子类的第一个位置，好处是首地址相同，可以直接做类型转换。子类型需要实现对应版本的接口函数，
`hw_module_t` 作为 module 的基类，放置于继承的具体模块类型 struct 子类的第一个位置，好处是首地址相同，可以直接做类型转换。基类定义了唯一一个接口函数。

`hw_module_methods_t.open`, 打开硬件设备。所有模块都需要实现该接口。不同类型的模块可以定义自己的接口函数，比如 camera, gralloc, hwcomposer 等等


## 模块加载
指定模块名称 **MODULE_ID**，加载时根据模块名称查找对应的动态库并加载，获取模块结构体对象地址，调用 **open device** 打开硬件设备，即可使用设备提供的功能。

路径查找，先根据 variant 确定动态库后缀名称，`<MODULE_ID>.variant.so`，按优先级依次是：
1. **ro.hardware**
2. **ro.product.board**
3. **ro.board.platform**
4. **ro.arch**

完整名称确定后，按照如下优先级查找对应路径：
1. **/odm/lib64/hw**
2. **/vendor/lib64/hw**
3. **/system/lib64/hw**

所有模块的 `hw_module_t.hal_api_version` 字段必须填入 `HARDWARE_HAL_API_VERSION`，该值目前取值是 **1.0**

```c++
>>> libhardware/include/hardware/hardware.h
/*
 * The current HAL API version.
 *
 * All module implementations must set the hw_module_t.hal_api_version field
 * to this value when declaring the module with HAL_MODULE_INFO_SYM.
 *
 * Note that previous implementations have always set this field to 0.
 * Therefore, libhardware HAL API will always consider versions 0.0 and 1.0
 * to be 100% binary compatible.
 *
 */
#define HARDWARE_HAL_API_VERSION HARDWARE_MAKE_API_VERSION(1, 0)

/**
 * Name of the hal_module_info
 */
#define HAL_MODULE_INFO_SYM         HMI

/**
 * Name of the hal_module_info as a string
 */
#define HAL_MODULE_INFO_SYM_AS_STR  "HMI"

/**
 * Get the module info associated with a module by id.
 *
 * @return: 0 == success, <0 == error and *module == NULL
 */
int hw_get_module(const char *id, const struct hw_module_t **module);
```


```c++
>>> hardware/libhardware/hardware.c

/** Base path of the hal modules */
#if defined(__LP64__)
#define HAL_LIBRARY_PATH1 "/system/lib64/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib64/hw"
#define HAL_LIBRARY_PATH3 "/odm/lib64/hw"
#else
#define HAL_LIBRARY_PATH1 "/system/lib/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"
#define HAL_LIBRARY_PATH3 "/odm/lib/hw"
#endif

static const char *variant_keys[] = {
    "ro.hardware",  /* This goes first so that it can pick up a different
                       file on the emulator. */
    "ro.product.board",
    "ro.board.platform",
    "ro.arch"
};

/**
 * Load the file defined by the variant and if successful
 * return the dlopen handle and the hmi.
 * @return 0 = success, !0 = failure.
 */
static int load(const char *id,
        const char *path,
        const struct hw_module_t **pHmi)
{
    int status = -EINVAL;
    void *handle = NULL;
    struct hw_module_t *hmi = NULL;

    handle = dlopen(path, RTLD_NOW);

    /* Get the address of the struct hal_module_info. */
    const char *sym = HAL_MODULE_INFO_SYM_AS_STR;
    hmi = (struct hw_module_t *)dlsym(handle, sym);

    /* Check that the id matches */
    if (strcmp(id, hmi->id) != 0) {
        ALOGE("load: id=%s != hmi->id=%s", id, hmi->id);
        status = -EINVAL;
        goto done;
    }

    hmi->dso = handle;

    /* success */
    status = 0;

    *pHmi = hmi;

    return status;
}
```

举个示例，gralloc 的默认实现。

1. 首先定义模块名称 MODULE_ID 是 “gralloc”，需要写入对应的 `hw_module_t.id` 字段，加载模块时会判断是否匹配。则对应动态库是 “gralloc.variant.so", variant 会在加载时搜索确定。
2. 其次是定义全局可见，并且名称固定为 `HAL_MODULE_INFO_SYM` 的结构体变量，模块加载时透过 `dlsym` 搜索该变量，获取到结构体对象地址。


```c++
>>> libhardware/include/hardware/gralloc.h
/**
 * The id of this module
 */
#define GRALLOC_HARDWARE_MODULE_ID "gralloc"


>>> libhardware/modules/gralloc/gralloc.cpp

static struct hw_module_methods_t gralloc_module_methods = {
        .open = gralloc_device_open
};

struct private_module_t HAL_MODULE_INFO_SYM = {
    .base = {
        .common = {
            .tag = HARDWARE_MODULE_TAG,
            .version_major = 1,
            .version_minor = 0,
            .id = GRALLOC_HARDWARE_MODULE_ID,
            .name = "Graphics Memory Allocator Module",
            .author = "The Android Open Source Project",
            .methods = &gralloc_module_methods
        },
        .registerBuffer = gralloc_register_buffer,
        .unregisterBuffer = gralloc_unregister_buffer,
        .lock = gralloc_lock,
        .unlock = gralloc_unlock,
    },
    .framebuffer = 0,
    .flags = 0,
    .numBuffers = 0,
    .bufferMask = 0,
    .lock = PTHREAD_MUTEX_INITIALIZER,
    .currentBuffer = 0,
};

/**
 * Every hardware module must have a data structure named HAL_MODULE_INFO_SYM
 * and the fields of this data structure must begin with hw_module_t
 * followed by module specific information.
 */
typedef struct gralloc_module_t {
    struct hw_module_t common;
    ...

```

`gralloc.default.so` 符号表，可以看到有 **HMI** 符号，获取到的就是 `private_module_t`结构体对象地址
```c++
Symbol table '.dynsym' contains 33 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
    24: 0000000000002a18   332 FUNC    GLOBAL DEFAULT   13 _Z14fb_device_openPK11hw_
    25: 0000000000002e94   200 FUNC    GLOBAL DEFAULT   13 _Z25gralloc_unregister_bu
    26: 0000000000004010   688 OBJECT  GLOBAL DEFAULT   15 HMI
    27: 0000000000003060   124 FUNC    GLOBAL DEFAULT   13 _Z12gralloc_lockPK16grall
    28: 00000000000030dc   116 FUNC    GLOBAL DEFAULT   13 _Z14gralloc_unlockPK16gra
    29: 0000000000002ff8   104 FUNC    GLOBAL DEFAULT   13 _Z15terminateBufferPK16gr
    30: 0000000000002504  1132 FUNC    GLOBAL DEFAULT   13 _Z20mapFrameBufferLockedP
    31: 0000000000002d68   300 FUNC    GLOBAL DEFAULT   13 _Z23gralloc_register_buff
    32: 0000000000002f5c   156 FUNC    GLOBAL DEFAULT   13 _Z9mapBufferPK16gralloc_m
```
