---
title: Android Camera 全栈通关：JNI 桥梁、跨进程 AIDL、BufferQueue 与硬件 DMA 零拷贝全链路深挖
date: 2026-06-24 12:00:00 +0800
categories: [Camera, Framework]
tags: [camera, android, hal3]
---

今天我们就**彻底撕开 Android 系统与芯片底层的面纱**。

本节课将是全网最硬核的 **Android Camera & RK3562 底层全链路通关指南**。我们将以一行控制命令（`openCamera`）和一帧图像数据（`1080P UYVY`）为线索，以**源码级、指针级、汇编/系统调用级、物理硬件寄存器级**的分步拆解，带你彻底看清整个 Camera 系统的脉络。

---

# 🚀 Android Camera 全栈通关：JNI 桥梁、跨进程 AIDL、BufferQueue 与硬件 DMA 零拷贝全链路深挖

---

## 🛠️ 第一部分：控制流的起点 —— JNI 桥梁与 C++ 对象重构（Java ➔ C++）

当应用层执行：
```java
mCameraDevice.createCaptureSession(Arrays.asList(previewSurface), ...);
```
这个 `previewSurface` 是怎么在 JNI 层被剥离，并转化为 C++ 的 `sp<IGraphicBufferProducer>` 强指针的？

### 1. JNI 层的底层脱壳：`android_view_Surface.cpp`
在 Android 框架中，Java 层的 `Surface` 对象只是一个空壳，它持有一个 C++ 层的 `long` 类型指针 `mNativeObject`。
*   **AOSP 源码路径**：`frameworks/base/core/jni/android_view_Surface.cpp`
*   **JNI C++ 核心执行源码**：

```cpp
#include <android/native_window/imageder.h>
#include <gui/Surface.h>
#include <android_runtime/android_view_Surface.h>

using namespace android;

// 1. JNI 方法签名映射
static jlong nativeCreateFromSurfaceTexture(JNIEnv* env, jclass clazz, jobject surfaceTextureObj) {
    // 获取 Java 层 SurfaceTexture 内部的 C++ GLConsumer 生产者对象
    sp<IGraphicBufferProducer> producer = SurfaceTexture_getProducer(env, surfaceTextureObj);
    if (producer == nullptr) {
        jniThrowException(env, "java/lang/IllegalArgumentException", "SurfaceTexture already released");
        return 0;
    }
    
    // 2. 实例化 C++ 层的 Surface 对象（继承自 ANativeWindow）
    sp<Surface> surface = new Surface(producer, true);
    if (surface == nullptr) {
        return 0;
    }
    
    // 3. 增加强引用计数，并将 C++ Surface 强指针转换为裸指针，作为 long 类型传回 Java 层保存
    surface->incStrong(&gSurfaceClassInfo);
    return reinterpret_cast<jlong>(surface.get());
}

// 4. 当 Framework 需要向下传递 IGraphicBufferProducer 时：
sp<IGraphicBufferProducer> android_view_Surface_getIGraphicBufferProducer(JNIEnv* env, jobject surfaceObj) {
    if (surfaceObj == nullptr) return nullptr;
    
    // 从 Java Surface 对象的 mNativeObject 字段中取出强行转换回来的 C++ Surface 指针
    sp<Surface> surface(reinterpret_cast<Surface*>(env->GetLongField(surfaceObj, gSurfaceClassInfo.mNativeObject)));
    if (surface == nullptr) return nullptr;
    
    // 提取核心：IGraphicBufferProducer (BufferQueue 的生产者接口)
    return surface->getIGraphicBufferProducer();
}
```

### 2. JNI 层的参数打包：`CameraMetadataNative` 的指针强转
你在 Java 层设置的 `CaptureRequest.set(CaptureRequest.CONTROL_AE_MODE, ...)`，是如何存进 C++ 内存的？
*   **AOSP 源码路径**：`frameworks/base/core/jni/android_hardware_camera2_CameraMetadata.cpp`
*   **核心转换代码**：

```cpp
static jlong CameraMetadata_isEmpty(JNIEnv* env, jobject thiz) {
    // 1. 提取 Java 对象内部持有的 C++ CameraMetadata 裸指针
    CameraMetadata* metadata = reinterpret_cast<CameraMetadata*>(env->GetLongField(thiz, gCameraMetadataClassInfo.mMetadataPtr));
    if (metadata == nullptr) {
        jniThrowNullPointerException(env, "Metadata is null");
        return JNI_TRUE;
    }
    // 2. 直接对 C++ 裸内存进行操作
    return metadata->isEmpty() ? JNI_TRUE : JNI_FALSE;
}
```
*   **原理**：Java 层和 C++ 层共享同一块 C++ 堆内存（`malloc` 分配的 `CameraMetadata`）。Java 层只持有它的 `jlong`（64位内存物理/虚拟地址）。JNI 通过 `reinterpret_cast` 强制类型转换直接读写此内存，**零内存拷贝，效率极高**。

---

## 🎛️ 第二部分：控制流的延伸 —— `cameraserver` 进程内部控制链与底层 AIDL 通信

现在，控制命令跨越 Binder 边界进入了系统的守护进程 `/system/bin/cameraserver`。

```text
[cameraserver 进程]
  CameraService.cpp 
    └──> 实例化 CameraDeviceClient.cpp (处理特定 App 的请求)
          └──> 拥有 Camera3Device.cpp (对接 HAL3 规范)
                └──> 唤醒 RequestThread 线程
                      └──> 调用 AIDL: ICameraDeviceSession::processCaptureRequest
```

### 1. `Camera3Device` 的高并发控制核心：`RequestThread` 运作环
在 `Camera3Device.cpp` 中，一旦 App 下发了 Request，就会唤醒内部的 `RequestThread`。
*   **AOSP 源码路径**：`frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp`
*   **`RequestThread::threadLoop` 核心细节源码**：

```cpp
bool Camera3Device::RequestThread::threadLoop() {
    status_t res;
    sp<CaptureRequest> nextRequest = nullptr;

    // 1. 线程阻塞等待：直到 App 下发新的 Request
    res = mRequestQueue.waitAndPop(&nextRequest);
    if (res != OK) return false; // 退出线程

    // 2. 准备底层的请求容器
    camera_capture_request_t halRequest;
    memset(&halRequest, 0, sizeof(halRequest));

    halRequest.frame_number = nextRequest->mResultExtras.frameNumber;
    
    // 3. 将 C++ CameraMetadata 绑定到 HAL 请求
    halRequest.settings = nextRequest->mSettingsList.begin()->metadata.getAndLock();

    // 4. 将输出 Buffer（GraphicBuffer）的句柄剥离出来放入 HAL 请求
    Vector<camera_stream_buffer_t> outBuffers;
    for (size_t i = 0; i < nextRequest->mOutputSurfaces.size(); i++) {
        camera_stream_buffer_t streamBuffer;
        // 获取底层的 GraphicBuffer 的 native_handle_t 指针
        streamBuffer.buffer = &(nextRequest->mOutputSurfaces[i]->getGraphicBuffer()->handle);
        streamBuffer.status = CAMERA_BUFFER_STATUS_OK;
        streamBuffer.acquire_fence = -1; // 暂无 Fence 锁
        streamBuffer.release_fence = -1;
        outBuffers.push_back(streamBuffer);
    }
    halRequest.num_output_buffers = outBuffers.size();
    halRequest.output_buffers = outBuffers.array();

    // 5. 核心跨进程：通过 AIDL 接口将控制命令与内存 fd 传递给 HAL 进程
    auto err = mInterface->processCaptureRequest(&halRequest);
    
    nextRequest->mSettingsList.begin()->metadata.unlock(halRequest.settings);
    return true; // 循环继续
}
```

### 2. 底层 AIDL 通信细节：Binder 传输控制包
当 `mInterface->processCaptureRequest(&halRequest)` 被调用时，底层使用的是 C++ 自动生成的 AIDL 代码。
*   **核心动作**：
    1. 将 `frame_number`、参数、`buffer_handle_t`（包含共享内存的 `dma_fd`）打包进 `Parcel`。
    2. 执行 `binder::Parcel::writeStrongBinder` 写入 Binder 驱动。
    3. 触发系统调用：`ioctl(binder_fd, BINDER_WRITE_READ, &bwr)`。
    4. 线程挂起，等待 `/vendor/bin/hw/android.hardware.camera.provider` 进程处理完毕后唤醒。

---

## 🌊 第三部分：数据流的生命通路 —— BufferQueue 与物理内存管理（Gralloc/ION）

在 Camera 系统中，如何做到一帧数百兆的图像帧在多进程间**“零拷贝（Zero-Copy）”**流转？答案是基于 **dma-buf** 机制的 **BufferQueue**。

### 1. Gralloc 的物理本质：ION 与 dma-buf 的内核分配
当你通过 `ImageReader` 申请内存时，底层的 **Gralloc（Graphics Allocator）** 模块开始工作。
*   **分配源头（内核空间）**：在 Linux 内核中，Gralloc 通过 `/dev/ion` 驱动在系统的 **ION 内存堆**（或者基于现代 DMA-BUF Heap）中开辟一块**物理连续**（或者由 IOMMU 映射成连续虚拟地址）的内存块。
*   **文件描述符（fd）化**：内核将这块物理内存的控制权包装成一个标准的 Linux **文件描述符（fd）**，称为 **`dma_fd`**。
*   **句柄包装（`buffer_handle_t`）**：
    在 C++ 中，这个 fd 以及它的 Stride（步长）、宽度、高度被封装进 `native_handle_t`。

```cpp
// 瑞芯微 Gralloc 底层的 Buffer 句柄结构体
typedef struct private_handle_t {
    int fd;           // 指向系统 ION 共享内存的 dma_fd <--- 零拷贝的核心！
    int size;         // 内存大小
    int width;        // 宽度
    int height;       // 高度
    int format;       // 图像格式（如 HAL_PIXEL_FORMAT_YCrCb_420_SP）
    uint64_t share_fd;// 共享文件描述符
    uintptr_t base;   // 本地进程的虚拟映射首地址
} private_handle_t;
```

### 2. BufferQueue 状态机的强力细节走读
BufferQueue 底层维护了一个数组，最大长度为 64（`BufferSlot`），用来缓存这些 `GraphicBuffer`。

```cpp
// AOSP 源码路径：frameworks/native/libs/gui/BufferQueueProducer.cpp
status_t BufferQueueProducer::dequeueBuffer(int* outSlot, sp<Fence>* outFence,
                                            uint32_t width, uint32_t height,
                                            PixelFormat format, uint64_t usage, ...) {
    Mutex::Autolock lock(mCore->mMutex);
    
    int found = BufferQueueCore::INVALID_BUFFER_SLOT;
    
    // 1. 寻找一个处于 FREE 状态的空闲槽位
    for (int i = 0; i < BufferQueueDefs::NUM_BUFFER_SLOTS; i++) {
        if (mCore->mSlots[i].mBufferState == BufferState::FREE) {
            found = i;
            break;
        }
    }
    
    if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
        return NO_MEMORY; // 队列满了，发生阻塞
    }

    // 2. 如果该槽位的 GraphicBuffer 还没有分配，或者规格变了，则重新向 Gralloc 申请内存
    if (mCore->mSlots[found].mGraphicBuffer == nullptr) {
        sp<GraphicBuffer> graphicBuffer = new GraphicBuffer(width, height, format, usage);
        mCore->mSlots[found].mGraphicBuffer = graphicBuffer;
    }

    // 3. 将该槽位的状态修改为 DEQUEUED（已借出）
    mCore->mSlots[found].mBufferState = BufferState::DEQUEUED;
    
    *outSlot = found; // 将槽位号（如 0）返回给 HAL
    *outFence = mCore->mSlots[found].mFence; // 返回 Fence 锁
    return OK;
}
```

*   **入队过程 (`queueBuffer`)**：
    当 HAL 往这块 Buffer 里写完数据后，调用 `queueBuffer(slot)`：
    1. 状态改变：`DEQUEUED` ➔ `QUEUED`（已写满待消费）。
    2. 触发回调：调用 `mConsumerListener->onFrameAvailable()` 唤醒屏幕渲染进程（`SurfaceFlinger`）。

---

## 🧱 第四部分：内核与硬件的终极交握 —— V4L2 驱动与 DMA 硬件传输

现在，我们跨越**用户空间（User Space）**与**内核空间（Kernel Space）**的边界，看看 **RK3562 的 CIF (Camera Interface)** 驱动，以及 **TP2856** 是如何直接把图像塞进这块内存的。

```text
[ 物理 Sensor / TP2856 ]
       │ 
       ▼ (MIPI CSI-2 信号: YUYV 帧数据)
[ RK3562 CIF 物理控制器 ] 
       │ 
       ▼ (解析 MIPI VC 虚拟通道，将数据拉入 FIFO 缓冲区)
[ DMA 控制器 (Direct Memory Access) ] 
       │ 
       ▼ (利用 IOMMU, 强制将数据写入 GraphicBuffer 对应的物理内存地址)
[ System RAM (物理 DDR 内存) ] 
```

### 1. HAL 层通过 ioctl 与内核握手
瑞芯微的 Camera HAL 进程（`android.hardware.camera.provider`）在工作时，会直接通过标准 Linux `ioctl` 控制内核设备节点 `/dev/video0`：

```cpp
// 瑞芯微 HAL 层控制驱动的核心代码段
// 源码对应：hardware/rockchip/camera/common/CameraHalDeviceOps.cpp

int CameraHalDevice::startStreaming() {
    enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    
    // 1. 设置 V4L2 视频采集格式：YUYV (TP2856输出格式)
    struct v4l2_format fmt;
    memset(&fmt, 0, sizeof(fmt));
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    fmt.fmt.pix.width = 1920;
    fmt.fmt.pix.height = 1080;
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV; 
    ioctl(mCameraFd, VIDIOC_S_FMT, &fmt);

    // 2. 告诉内核，我们要使用 DMABUF 零拷贝模式
    struct v4l2_requestbuffers req;
    memset(&req, 0, &req);
    req.count = 4; // 请求 4 个环形缓冲区
    req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    req.memory = V4L2_MEMORY_DMABUF; // 零拷贝的核心映射模式！
    ioctl(mCameraFd, VIDIOC_REQBUFS, &req);

    // 3. 将从 BufferQueue 中申请出来的 dma-fd，直接绑定到 V4L2 的缓冲区槽位上
    for (int i = 0; i < 4; i++) {
        struct v4l2_buffer buf;
        memset(&buf, 0, sizeof(buf));
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_DMABUF;
        buf.index = i;
        buf.m.fd = mGraphicBuffers[i].getDmaFd(); // 将底层 Gralloc 内存的 fd 塞给内核！
        ioctl(mCameraFd, VIDIOC_QBUF, &buf); // 入队 V4L2 驱动队列
    }

    // 4. 下发开始指令，开启 MIPI CSI-2 接收器
    ioctl(mCameraFd, VIDIOC_STREAMON, &type);
    return 0;
}
```

### 2. RK3562 内核驱动层：`rk_cif` 的 DMA 直接写内存
当 V4L2 开启后，内核的 CIF 驱动开始工作。
*   **瑞芯微内核源码路径**：`kernel/drivers/media/platform/rockchip/cif/`
*   **硬件中断执行逻辑（`dev.c` 伪代码）**：

```c
// 当一帧 MIPI 数据传输完毕，硬件会向 CPU 发送一个硬件中断，触发驱动的中断处理函数
static irqreturn_t rkcif_irq_handler(int irq, void *ctx) {
    struct rkcif_device *cif_dev = ctx;
    
    // 1. 读取 RK3562 CIF 控制器的硬件寄存器，确定当前的传输状态
    u32 stat = rkcif_read_reg(cif_dev, CIF_INTSTAT);
    
    if (stat & FRAME_END_INT) {
        // 2. 清除中断标志位
        rkcif_write_reg(cif_dev, CIF_INTCL, FRAME_END_INT);
        
        // 3. 获取 V4L2 队列中当前的活动缓冲区
        struct rkcif_buffer *active_buf = cif_dev->active_buf;
        
        // 4. 获取由 Gralloc 传进来的物理内存地址 (已经过 IOMMU 映射)
        unsigned long phys_addr = vb2_dma_contig_plane_dma_addr(&active_buf->vb.vb2_buf, 0);
        
        // 5. 将这帧数据的物理首地址，直接写入 RK3562 CIF 的 DMA 控制器目标寄存器
        rkcif_write_reg(cif_dev, CIF_DMA_ADDR_Y, phys_addr);
        rkcif_write_reg(cif_dev, CIF_DMA_ADDR_UV, phys_addr + (1920 * 1080));

        // 6. 唤醒正在阻塞等待的用户空间 HAL 进程：数据已写完！
        vb2_buffer_done(&active_buf->vb.vb2_buf, VB2_BUF_STATE_DONE);
    }
    return IRQ_HANDLED;
}
```
*   **极其震撼的物理机制**：
    1. 在整个图像传输和处理的过程中，**CPU 完全没有去执行任何 `memcpy` 指令**！
    2. RK3562 的 CIF 控制器自带一个 **DMA 引擎**。它直接通过物理总线把来自 MIPI 接口的数据，一个像素一个像素地搬运到我们通过 `GraphicBuffer` 申请的那块物理 DDR 内存里。
    3. 数据搬完了，发一个硬件中断通知 CPU。CPU 只需要通知上层应用：“面已经煮好了，端走去用吧！”

---

## 📺 第五部分：图像的最终绽放 —— 屏幕显示渲染链路

图像已经写入了这块 ION 内存，并且也已经通过 `queueBuffer` 还给了系统。屏幕渲染服务 **`SurfaceFlinger`** 此时被唤醒。

```text
[ SurfaceFlinger 进程 ] 
  ├──> 调用 BufferQueueConsumer::acquireBuffer() 拿到带有 dma_fd 的 GraphicBuffer
  ├──> 将其转换为 GPU 可以识别的 OES 外部纹理 (External OES Texture)
  └──> 或者是直接将该图层直接下发给系统的 DPU (视频显示处理器)
```

### 1. OpenGL ES 的纹理无缝绑定：OES 扩展
如果你的 DVR 画面需要做一些实时的滤镜、360 度环视畸变矫正算法，GPU 必须能够读取这块内存。
在 Native C++ 算法中，我们使用 `EGL` 扩展来绑定底层 Buffer：

```cpp
#include <EGL/eglext.h>
#include <GLES2/gl2.h>
#include <GLES2/gl2ext.h>

GLuint bindGraphicBufferToTexture(sp<GraphicBuffer> graphicBuffer) {
    // 1. 获取当前显示环境
    EGLDisplay dpy = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    
    // 2. 定义绑定参数
    EGLint attrs[] = {
        EGL_IMAGE_PRESERVED_KHR, EGL_TRUE,
        EGL_NONE
    };

    // 3. 将系统的 GraphicBuffer (即底层的 dma_fd) 直接包装成 EGLImage
    EGLImageKHR image = eglCreateImageKHR(
        dpy,
        EGL_NO_CONTEXT,
        EGL_NATIVE_BUFFER_ANDROID, // 核心参数：告诉 EGL 这是一块 Android 原生内存
        (EGLClientBuffer)graphicBuffer->getNativeBuffer(),
        attrs
    );

    // 4. 创建并绑定 OpenGL 外部纹理
    GLuint textureId;
    glGenTextures(1, &textureId);
    glBindTexture(GL_TEXTURE_EXTERNAL_OES, textureId);

    // 5. 执行纹理关联操作：将 EGLImage 映射到 OES 纹理上
    glEGLImageTargetTexture2DOES(GL_TEXTURE_EXTERNAL_OES, image);
    
    // 此时，GPU 就可以在渲染管线中直接读取这块物理内存的内容了，完全不需要 CPU 内存上传！
    return textureId;
}
```

### 2. 车机首选的硬件图层叠加 (HWC overlay)
如果你的倒车影像、DVR 不需要进行 3D 渲染，只是纯显示在屏幕上，那甚至都不需要用到 GPU（这可以为车机节省极大的电能和热能，降低系统负荷）。
*   **HWC (Hardware Composer) 模式**：
    1. `SurfaceFlinger` 发现这一路流是全屏或者矩形区域直接显示。
    2. 它直接把这帧 `GraphicBuffer` 对应的内存物理地址发送给系统的 **DPU（Display Processor Unit / 瑞芯微 VOP 控制器）**。
    3. DPU 硬件在对 LCD 屏进行逐行扫描输出时，**直接读取这块物理内存**，把色彩打在屏幕对应的像素点上。

---


如果觉得今天的内容听得足够过瘾、已经完全消化，请回复我：“**老师，大纲和第一课内容已完美吸收！我们开始第二阶段的第一课：深入解构 Java Framework 层的 CameraManager 源码，探秘相机的发现与状态监听细节！**”
