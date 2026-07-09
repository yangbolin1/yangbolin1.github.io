---
layout: post
title: "深入底层：Android Camera Framework 源码硬核解析（四）—— 全链路解剖 Session 家族与底层创建机制 (基于 Android 16)"
date: 2026-07-10 10:00:00 +0800
categories: [Android底层, 源码解析]
tags: [Camera2, Framework, Session, C++, 架构设计]
---

> **📝 导读**
> 从 Camera1 迁移到 Camera2 的开发者常有一个痛点：为什么不能直接 `startPreview()`，非要繁琐地配置 `Surface` 并创建一个 `Session`（会话）？
> 
> 如果你打开 Android 源码的 `frameworks/base/.../camera2/` 目录，你会发现里面竟然存在着四五种不同的 Session 类。它们究竟有何不同？底层到底是怎么实现的？
> 
> 今天，我们将化身“高级水管工”，翻开最新的 **Android 16 源码**，从宏观的**四大 Session 家族形态**，一路向下击穿 Framework，直达底层的 **C++ cameraserver 进程**，为你揭开“搭建图像数据流水线”的终极秘密！

---

## 🤔 第一部分：核心拷问 —— 为什么必须有 Session？

**大白话比喻：**
`CameraDevice`（相机硬件）就像是一个巨大的水库，里面有源源不断的水（图像数据）。
你的 App 里的 `SurfaceView` 或 `ImageReader`，就像是你家里的各种水龙头和洗衣机。
**`Session` (会话)，就是连接水库和你家水龙头的那套“地下自来水管道系统”。**

**为什么不能即插即用？**
因为高清图像（如 4K/60fps）的数据量极其庞大！在通水之前，底层的 C++ (`cameraserver`) 必须提前知道：
1. 你有几个水龙头？
2. 每个水龙头管径多大（分辨率、数据格式是什么）？

底层需要根据这些信息，在系统的内存中精准分配 `GraphicBuffer`（共享物理内存）。**Session 的创建过程，实质上就是底层硬件分配内存、搭建流水线的握手协商过程。如果不建 Session 直接通水，系统内存直接崩溃！**

理解了前提，我们先来看看 Android 16 源码中，为了应对不同的业务场景，演化出了哪四大王牌“管道”。

---

## 🚰 第二部分：Session 家族的四大王牌形态

为了应对日常预览、240fps高刷、计算摄影和后台处理，Google 在 Framework 层采用了**策略模式**，派生出了四种极其特殊的 Session。

### 1. 标准自来水管：`CameraCaptureSessionImpl`
这是绝对的主力军，平时 90% 的预览和拍照都在用它。

**💻 源码剖析：** 当你调用 `setRepeatingRequest`（开启连续预览）时：
```java
// 源码位置：CameraCaptureSessionImpl.java

@Override
public int setRepeatingRequest(CaptureRequest request, CaptureCallback callback, Handler handler) {
    // 💡 Android 16 强推 Executor，对老旧的 Handler 进行封装降级
    Executor executor = (handler == null) ? mStateExecutor : new HandlerExecutor(handler);

    synchronized (mDeviceImpl.mInterfaceLock) {
        checkNotClosed();
        // 甩手掌柜：将请求转交给大老板 CameraDeviceImpl 去排队
        return mDeviceImpl.submitCaptureRequest(
                new CaptureRequest[] { request }, 
                callback, executor, true /* isRepeating: 告诉底层这是连拍！*/); 
    }
}
```

### 2. 高铁专用管：`CameraConstrainedHighSpeedCaptureSessionImpl`
**痛点**：拍 240fps 慢动作时，每帧处理时间只有 4 毫秒！如果一帧帧发请求，Java 和 C++ 频繁的 Binder IPC 跨进程通信绝对会导致 CPU 爆表。
**解法**：开启 `BatchedOutputs` (批量发车机制)。

**💻 源码剖析：**
```java
// 源码位置：CameraConstrainedHighSpeedCaptureSessionImpl.java

@Override
public List<CaptureRequest> createHighSpeedRequestList(CaptureRequest request) {
    // 💡 核心机制：一次性打包多帧（比如 8 帧）
    int requestListSize = mHighSpeedRequestSize; 
    List<CaptureRequest> requestList = new ArrayList<>(requestListSize);

    for (int i = 0; i < requestListSize; i++) {
        CaptureRequest.Builder builder = new CaptureRequest.Builder(request);
        builder.set(CaptureRequest.CONTROL_AE_TARGET_FPS_RANGE, mHighSpeedFpsRange); // 锁死高帧率
        requestList.add(builder.build());
    }
    return requestList; // 强制返回 List，底层 C++ 收到后一次性闷头抓取 8 帧！降低 8 倍 IPC 频次！
}
```

### 3. 魔法滤镜管：`CameraExtensionSessionImpl`
**痛点**：第三方 App 拍夜景噪点多，无法使用手机原厂自带的“超级夜景”等闭源算法。
**解法**：Extension OEM 算法代理。

**💻 源码剖析：**
```java
// 源码位置：CameraExtensionSessionImpl.java
private void initialize() {
    // 1. 获取厂商预埋在系统里的 Extension 算法进程代理！
    mExtensionProxy = CameraManagerGlobal.get().getCameraExtensionProxy();
    // 2. 告诉厂商：App 请求开启超级夜景 (EXTENSION_NIGHT)
    mExtensionProxy.initializeSession(mExtensionType, mOutputSurfaces);
    // 💡 结果：数据流不再直达 App，而是 Sensor -> mExtensionProxy(提亮/降噪) -> App
}
```

### 4. 离线接力棒：`CameraOfflineSession`
**痛点**：拍 1 亿像素照片需要处理 3 秒，用户如果按完快门瞬间把 App 杀后台，照片就会丢失。
**解法**：离线模式，实现进程控制权交接。

**💻 源码剖析：**
```java
// 源码位置：CameraDeviceImpl.java 
@Override
public CameraOfflineSession switchToOffline(...) {
    // 🔥 核心爆发点：告诉底层的 C++ cameraserver，我 App 不管了，你接管吧！
    ICameraOfflineSession remoteOfflineSession = 
            mRemoteDevice.switchToOffline(mInterface, offlineStreamIds, /*out*/ offlineRequestInfo);
            
    // 在 Java 层创建一个离线壳子返回
    return new CameraOfflineSessionImpl(mCameraId, remoteOfflineSession, executor, callback);
}
```
*执行此方法后，App 安心死掉，C++ 系统进程会在后台默默把算法跑完并写入系统图库。*

---

## 🔨 第三部分：手撕 Session 创建的底层全链路

上面我们见识了 Session 家族的四大形态。那么，**当我们在 App 里调用 `createCaptureSession` 时，这根水管在底层到底是怎么一步步铺设出来的？**

让我们以标准 Session 为例，顺着调用链，一路击穿 Java 层，深入 C++ HAL 层！

### 👣 Step 1：App 层的发起
我们在应用层通常会这么写现代 Android API：
```java
// 将 Surface 包装，指定 Session 类型并下发请求
SessionConfiguration sessionConfig = new SessionConfiguration(
    SessionConfiguration.SESSION_REGULAR, outputs, executor, stateCallback);
mCameraDevice.createCaptureSession(sessionConfig);
```

### 👣 Step 2：Java 层向底层的跨进程呼叫 (`CameraDeviceImpl.java`)
调用最终会走到 Framework 的 `CameraDeviceImpl`。此时，Java 层的代理人并没有立刻 `new` 一个 Session 对象，而是带着你的 `Surface` 去底层申请内存了。

```java
// 源码位置：CameraDeviceImpl.java

public boolean configureStreamsChecked(InputConfiguration inputConfig,
        List<OutputConfiguration> outputs, int operatingMode) {
    try {
        // 1. 跨进程通话开始：告诉底层 C++，我要开始配置流了
        mRemoteDevice.beginConfigure();

        // 2. 遍历你传进来的每一个 Surface
        for (OutputConfiguration outConfig : outputs) {
            // 告诉底层：给我建一条新的输出水管
            int streamId = mRemoteDevice.createStream(outConfig);
            mConfiguredOutputs.put(streamId, outConfig); 
        }

        // 3. 告诉底层 C++：我的水管需求提完了，你去干活（分配物理内存）吧！
        mRemoteDevice.endConfigure(operatingMode, null, ...);
        return true;
    } catch (RemoteException e) {
        return false;
    }
}
```
> **📌 敲黑板：** `beginConfigure`、`createStream`、`endConfigure` 是三个极其核心的 IPC Binder 调用。你的 Java `Surface` 就在这一步被送到了底层的 C++ `cameraserver` 进程。

### 👣 Step 3：C++ 底层接管，搭建硬件流水线 (`Camera3Device.cpp`)
现在，代码执行流越过了语言边界，来到了 C++ 层。底层的 `Camera3Device` 收到 `endConfigure` 信号后，开始真正的体力劳动。

```cpp
// 源码位置：frameworks/av/services/camera/libcameraservice/device3/Camera3Device.cpp

binder::Status Camera3Device::endConfigure(int operatingMode, ...) {
    Mutex::Autolock il(mInterfaceLock);

    // 1. 整理 App 传过来的所有输出流 (Streams)
    camera_stream_configuration_t streamList;
    streamList.num_streams = mOutputStreams.size();
    streamList.streams = mOutputStreams.data();

    // 2. 🚨 终极调用：向 HAL 硬件抽象层下达指令！
    // mHal3Device 代表的是具体手机厂商（高通/联发科等）的硬件驱动接口
    status_t res = mHal3Device->ops->configure_streams(mHal3Device, &streamList);

    if (res == OK) {
        // 3. 硬件配置成功！开始为每个流分配 GraphicBuffer 共享物理内存
        for (size_t i = 0; i < mOutputStreams.size(); i++) {
            camera3_stream_t *dstStream = mOutputStreams[i];
            
            // 通知 Android 的图形系统 (BufferQueue) 去分配真正的连续物理内存空间！
            res = dstStream->finishConfiguration();
        }
    }
    return binder::Status::ok();
}
```

### 💡 深度剖析：底层的“商务谈判”
在 C++ 层，`mHal3Device->ops->configure_streams` 实际上是一场 **Android 系统服务与底层 ISP 硬件芯片的商务谈判**！

*   **系统告诉硬件**：“App 需要一个 1080P 的预览流（YUV格式）和一个 4K 的拍照流（JPEG格式）。”
*   **硬件评估后回答**：“没问题。但为了保证不卡顿，1080P 的流你需要给我分配 **3 块**内存轮换池，4K 的流你需要分配 **2 块**内存，而且必须是底层连续的 DMA 内存。”
*   **谈判完成后**：C++ 层调用 `finishConfiguration`，严格按照硬件芯片提出的要求，向 Android 系统申请了真正的物理内存块（`GraphicBuffer`）。

当这一切在底层全部就绪，C++ 返回 OK，Java 层才会最终执行 `new CameraCaptureSessionImpl(...)`，并触发你 App 中的 `onConfigured` 回调！

---

## 🎯 架构总结：重新认识 Session

经过这番宏观到微观、Java 到 C++ 的全链路源码巡礼，我们可以给 Session 下一个最硬核的定义了：

1. **在宏观架构上（Java 层）**：它是 Google 采用**策略模式**的典范。通过拆分标准、高速、OEM 扩展、离线四大 Session，完美解耦了日益复杂的影像业务需求。它是你下发 `CaptureRequest` 的**队列管理器**。
2. **在微观本质上（C++ 层）**：创建 Session 是一次**深度的硬件资源分配行为**。它将 App 层的 `Surface` 转化为了硬件认可的数据流，并完成了最消耗系统性能的动作——**GraphicBuffer 共享内存的协商与分配**。

这也完美解释了：**为什么 `createCaptureSession` 是一个极其耗时的异步操作（通常需要几百毫秒）？** 因为它在底层不仅要跨进程，还要和硬件驱动讨价还价、申请海量的连续物理内存！

掌握了这些，以后当你在项目中写下 `createCaptureSession` 时，你的视野早已穿透了屏幕，直达主板上那颗飞速运转的 ISP 芯片！

---
> **💬 互动环节**
> 顺着这条源码链路追踪下来，你是否有一种“任督二脉被打通”的感觉？你有没有遇到过 Session 创建失败（onConfigureFailed）的坑？往往是因为分辨率配置超出了硬件上限！欢迎在评论区分享你的踩坑经验！
```
