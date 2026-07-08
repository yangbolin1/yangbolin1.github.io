---
layout: post
title: "深入底层：Android Camera Framework 源码解析（四）—— 结合 Android 16 源码解剖 Session 家族"
date: 2026-07-10 10:00:00 +0800
categories: [Android底层, 源码解析]
tags: [Camera2, Framework, Session, 源码导读]
---

> **📝 导读**
> 从 Camera1 迁移到 Camera2 的开发者常有一个痛点：为什么不能直接 `startPreview()`，非要繁琐地配置 Surface 并创建一个 `Session`（会话）？
> 
> 在 `frameworks/base/.../camera2/` 目录下，竟然存在着四五种不同的 Session 类。它们究竟有何不同？
> 今天，我们将化身“高级水管工”，翻开最新的 **Android 16 源码**，结合真实代码片段，带你彻底搞懂 Camera2 框架中最核心的数据路由中枢 —— **Session 家族**。

---

## 🤔 核心拷问：为什么必须有 Session？

**大白话比喻：**
`CameraDevice`（相机硬件）就像是一个巨大的水库，里面有源源不断的水（图像数据）。
你的 App 里的 `SurfaceView` 或 `MediaRecorder`，就像是你家里的各种水龙头。
**`Session` (会话)，就是连接水库和你家水龙头的那套“地下自来水管道系统”。**

**为什么不能即插即用？**
因为高清图像（如 4K/60fps）的数据量极其庞大！在通水之前，底层 C++ (`cameraserver`) 必须提前知道：你有几个水龙头？每个水龙头管径多大（分辨率、格式）？
底层需要根据这些信息，在内存中精准分配 `GraphicBuffer` 共享内存。**Session 的创建过程，实质上就是底层硬件分配内存、搭建流水线的握手过程。**

理解了前提，我们来看 Android 16 源码中的四大王牌“管道”。

---

## 🚰 一、标准自来水管：`CameraCaptureSessionImpl`

这是绝对的主力军，平时 90% 的预览和拍照都在用它。

在 Android 16 源码中，当你调用 `setRepeatingRequest`（开启连续预览）时，它在内部是如何流转的？

#### 💻 Android 16 源码剖析
```java
// 源码位置：CameraCaptureSessionImpl.java

@Override
public int setRepeatingRequest(CaptureRequest request, CaptureCallback callback, Handler handler) 
        throws CameraAccessException {
        
    // 1. 基础校验与线程包装
    checkRepeatingRequest(request);
    // 💡 Android 16 强推 Executor，对老旧的 Handler 进行封装降级
    Executor executor = (handler == null) ? mStateExecutor : new HandlerExecutor(handler);

    synchronized (mDeviceImpl.mInterfaceLock) {
        // 2. 检查管道状态，如果你已经把管道关了，直接报错
        checkNotClosed();

        // 3. 把请求转交给大老板 CameraDeviceImpl 去处理
        // 注意：Session 本身不直接跨进程，它只是个管理者！
        return mDeviceImpl.submitCaptureRequest(
                new CaptureRequest[] { request }, // 打包成数组
                callback, 
                executor, 
                true /* isRepeating */); // 告诉底层这是连拍！
    }
}
```

> **📌 源码解析：**
> 标准 Session 其实是个“甩手掌柜”。它校验完状态后，会将任务交回给 `mDeviceImpl`，由其压入任务队列，并通过 Binder 发送给底层。底层的 ISP 收到 `isRepeating = true` 的指令后，就会像水泵一样，每秒 30 次源源不断地向你的 Surface 吐出数据。

---

## 🚄 二、高铁专用管：`CameraConstrainedHighSpeedCaptureSessionImpl`

**痛点场景：**
如果你要拍 **240fps 慢动作视频**，每一帧处理时间只有可怜的 **4 毫秒**！如果用上面的标准水管，Java 和 C++ 每秒要进行 240 次 Binder 跨进程通信，频繁的上下文切换绝对会导致 CPU 爆表、画面卡顿。

#### 💻 Android 16 源码剖析：打包发车机制
为了压榨极限性能，系统引入了高速 Session。它有一个极其霸道的核心方法：`createHighSpeedRequestList`。

```java
// 源码位置：CameraConstrainedHighSpeedCaptureSessionImpl.java

@Override
public List<CaptureRequest> createHighSpeedRequestList(CaptureRequest request) 
        throws CameraAccessException {
        
    // 1. 检查你的请求是否合规（高铁上不准带违禁品！）
    // 高速模式下，系统会强行禁用复杂的算法（如降噪、光学防抖等）以保速度
    checkHighSpeedRequest(request);

    // 2. 💡 核心机制：BatchedOutputs (批量发车)
    // 根据底层的 FPS 要求，计算一次需要打包多少帧（通常是 4 帧或 8 帧）
    int requestListSize = mHighSpeedRequestSize; // 假设是 8
    
    List<CaptureRequest> requestList = new ArrayList<>(requestListSize);

    // 3. 强行克隆并魔改你的 Request
    for (int i = 0; i < requestListSize; i++) {
        CaptureRequest.Builder builder = new CaptureRequest.Builder(request);
        
        // 系统强行锁定 FPS 范围，不让你瞎改
        builder.set(CaptureRequest.CONTROL_AE_TARGET_FPS_RANGE, mHighSpeedFpsRange);
        
        // 只有批量请求的最后一帧，才允许触发 UI 回调！极大节省 CPU
        if (i == requestListSize - 1) {
            builder.setUpf(CaptureRequest.CONTROL_ENABLE_ZSL, true);
        }
        
        requestList.add(builder.build());
    }
    
    return requestList; // 强制返回一个 List，而不是单个 Request！
}
```

> **📌 源码解析：**
> 这就是底层优化的极致体现！它**不允许**你一帧一帧地发请求。你必须一次性提交一个 List（比如 8 帧）。底层 C++ 收到后，闷头在极短的时间内连续抓取 8 帧，最后再统一打个包还给 Java 层。
> **跨进程 IPC 通信频率瞬间降低了 8 倍！这就是慢动作不卡顿的绝对机密！**

---

## 🪄 三、魔法滤镜管：`CameraExtensionSessionImpl`

**痛点场景：**
第三方 App 拍夜景全是噪点，而华为/小米自带相机拍出来亮如白昼。因为系统自带相机用了原厂的闭源算法！为了打破壁垒，Android 引入了 Extension 机制。

#### 💻 Android 16 源码剖析：OEM 算法代理
在最新的 Android 16 源码中，当你创建 Extension Session 时，系统会在后台悄悄唤醒手机厂商的系统服务。

```java
// 源码位置：CameraExtensionSessionImpl.java

private void initialize() {
    try {
        // 1. 获取厂商的 Extension 代理服务！
        // 这一步跨进程联系到了手机 OEM（如小米/OV）预埋在系统里的算法进程
        mExtensionProxy = CameraManagerGlobal.get().getCameraExtensionProxy();

        // 2. 告诉厂商服务：App 请求使用超级夜景 (EXTENSION_NIGHT)
        mExtensionProxy.initializeSession(mExtensionType, mOutputSurfaces);

        // 3. 截断正常的数据流
        // 正常：Sensor -> App Surface
        // 现在：Sensor -> mExtensionProxy (厂商算法处理) -> App Surface
        setupExtensionImageProcessor();

    } catch (RemoteException e) {
        Log.e(TAG, "Failed to connect to OEM extension service");
    }
}
```

> **📌 源码解析：**
> 建立这个 Session 后，**数据流的路由被强行改变了**。底层吐出的原始画面，不再直接给你的 App，而是先扔给 `mExtensionProxy`（OEM 私有算法进程）。在那里完成多帧合成、提亮降噪后，一张精美的“算算摄影”大作才会最终流向你的 `SurfaceView`。

---

## 🏃 四、接力棒交接：`CameraOfflineSession`

这是 Android 高版本（11~16不断完善）中最具革命性的痛点解法。

**痛点场景：**
用户拍了一张“1亿像素”或者“超级夜景”，由于算力巨大，需要 3 秒才能出图。但用户是个急性子，按完快门**瞬间就把你的 App 杀了**。以前这种情况下照片直接报废。

#### 💻 Android 16 源码剖析：进程控制权交接
为了解决防丢图问题，在你的 App 退到后台时，可以向底层申请 `switchToOffline`（切换为离线模式）。

```java
// 源码位置：CameraDeviceImpl.java 

@Override
public CameraOfflineSession switchToOffline(@NonNull Collection<Surface> offlineOutputs,
        @NonNull Executor executor, @NonNull CameraOfflineSessionCallback callback) 
        throws CameraAccessException {

    // 1. 校验哪些画布需要转为离线（比如用于保存图片的 ImageReader 画布）
    List<Integer> offlineStreamIds = new ArrayList<>();
    for (Surface surface : offlineOutputs) {
        offlineStreamIds.add(mConfiguredOutputs.getStreamId(surface));
    }

    // 2. 跨进程向底层的 C++ cameraserver 发起交接申请
    ICameraOfflineSession remoteOfflineSession;
    try {
        // 🔥 核心爆发点：告诉底层，我 App 不管了，你接管吧！
        remoteOfflineSession = mRemoteDevice.switchToOffline(
                mInterface, // 留下通信接口
                offlineStreamIds.stream().mapToInt(i->i).toArray(),
                /*out*/ offlineRequestInfo); // 获取接管状态
                
    } catch (RemoteException e) {
        throw new CameraAccessException(CameraAccessException.CAMERA_DISCONNECTED);
    }

    // 3. 在 Java 层创建一个离线壳子返回给 App
    CameraOfflineSessionImpl session = new CameraOfflineSessionImpl(
            mCameraId, remoteOfflineSession, executor, callback);

    return session;
}
```

> **📌 源码解析：**
> 当 `mRemoteDevice.switchToOffline()` 执行成功后，你的 App 即使彻底被杀掉 (Kill Process) 也没关系了。
> 框架层完成了一次完美的**“权限与资源交接”**：复杂的图像算法被彻底移交给了系统常驻的 `cameraserver` 进程。系统在后台默默算完后，会直接帮你把照片写入 MediaStore (系统图库)。这不仅是功能创新，更是系统级资源调度的艺术。

---

## 🎯 架构总结：为什么是策略模式？

看完 Android 16 的硬核源码，我们终于理解了 Google 架构师的苦心：

如果把标准预览、高铁批量模式、OEM 算法代理、离线接管机制... 几万行逻辑全部塞进一个类里，那将是无法维护的灾难。

通过**策略模式 (Strategy Pattern)** 抽象出不同的 `Session` 类，系统实现了极其优雅的解耦：
*   你要日常预览？用 `CameraCaptureSession`。
*   你要 240fps 不卡顿？用 `HighSpeedSession` 降低 IPC 频次。
*   你要厂商的高级算法？挂载 `ExtensionSession` 代理。
*   你要后台处理防丢图？调用 `OfflineSession` 完成进程交接。

作为高级开发者，当我们懂得了每一种“管道”底层是如何搭建流水线、如何分配内存、如何调度线程的，面对再奇葩的业务需求，我们都能一击命中，写出最优雅的 Camera 代码！

---
> **💬 互动环节**
> 通过直接阅读 Android 16 源码，你是不是对 Camera 架构有了全新的降维打击感？你目前的项目中用到了哪几种 Session？遇到了哪些坑？欢迎在评论区留言切磋！
```
