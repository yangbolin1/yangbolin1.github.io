```yaml
---
layout: post
title: "深入底层：Android Camera Framework 源码硬核解析（二）—— 揭秘 CameraDeviceImpl 与跨进程回调"
date: 2026-07-08 10:00:00 +0800
categories: [Android底层, 源码解析]
tags: [Camera2, Framework, Binder, 性能优化]
---

> **📝 前言**
> 在上一篇文章中，我们剖析了 `CameraManager.openCamera()` 的跨进程跃迁之旅。当指令到达 C++ 层（`cameraserver`），底层硬件成功通电后，真正的重头戏才刚刚开始：
> **底层硬件出图了、对焦成功了、报错了，系统是如何通知我们 App 的？**
> 
> 今天，我们将视线转移到 `frameworks/base/core/java/android/hardware/camera2/impl/CameraDeviceImpl.java`，深度解剖 App 留在底层的“卧底接线员” —— `CameraDeviceCallbacks`，并一探 Android 11 引入的神秘特性 `OfflineSession`。

---

## 🏗️ 核心架构：CameraDeviceImpl 到底是什么？

我们在 App 层调用 `cameraManager.openCamera` 成功后，回调里会塞给我们一个 `CameraDevice` 对象。但稍微有经验的开发者都知道，`CameraDevice` 只是一个抽象类/接口。

真正干活的幕后黑手，是 **`CameraDeviceImpl`**。

它不仅负责维护当前相机的状态、管理 Request 请求队列，更重要的是，它内部包含了一个继承自 `ICameraDeviceCallbacks.Stub` 的 Binder 服务端实现：**`CameraDeviceCallbacks`**。

你可以把它理解为 **App 留在底层的“卧底接线员”**。因为底层的 C++ 服务无法直接调用 Java 的普通方法，它只能通过 Binder IPC 通信，拨打这个接线员的“专线电话”。

---

## 👣 源码高能拆解：卧底接线员的日常

当底层 C++ 线程通过 Binder 打来电话时，接线员 `CameraDeviceCallbacks` 会执行一系列极其严密的安保和分发操作。我们重点拆解其中最核心的两个回调。

### 🎬 场景一：快门按下的瞬间 (`onCaptureStarted`)

当感光元件 (Sensor) 真正开始收集光子（曝光开始）的那一瞬间，底层会立刻跨进程调用此方法。我们的 App 通常依靠这个回调来播放“咔嚓”的快门声。

```java
// 源码位置：CameraDeviceImpl.java 的内部类 CameraDeviceCallbacks

@Override
public void onCaptureStarted(final CaptureResultExtras resultExtras, final long timestamp) {
    int requestId = resultExtras.getRequestId();           
    final long frameNumber = resultExtras.getFrameNumber(); 

    synchronized(mInterfaceLock) { 
        // 🚨 必须加锁：底层可能有多个 C++ 线程并发打来电话
        if (mRemoteDevice == null) return; 

        // 🌟 核心拦截：Android 11+ 的离线会话 (Offline Session) 介入
        if (mOfflineSessionImpl != null) {
            // 如果处于离线状态，将回调转交给后台 Session 继续处理
            mOfflineSessionImpl.getCallbacks().onCaptureStarted(resultExtras, timestamp);
            return;
        }

        // 找到 App 当初下发的原始请求
        final CaptureCallbackHolder holder = CameraDeviceImpl.this.mCaptureCallbackMap.get(requestId);
        if (holder == null) return; 

        // 🔥 核心安全机制：脱下系统权限马甲
        final long ident = Binder.clearCallingIdentity(); 
        try {
            // 切回 App 当初传入的 Executor 线程
            holder.getExecutor().execute(() -> {
                // 💡 针对 120fps/240fps 慢动作的批量回调处理
                if (holder.hasBatchedOutputs()) {
                    for (int i = 0; i < holder.getRequestCount(); i++) {
                        holder.getCallback().onCaptureStarted(...); 
                    }
                } else {
                    // 正常单帧回调给 App 开发者
                    holder.getCallback().onCaptureStarted(
                        CameraDeviceImpl.this, request, timestamp, frameNumber);
                }
            });
        } finally {
            // 穿回系统马甲
            Binder.restoreCallingIdentity(ident); 
        }
    }
}
```

> **📌 源码硬核解析：**
> 
> 1. **神秘的 `mOfflineSessionImpl` 是什么？**
>    现在的手机拍照动辄 1 亿像素，或者进行多帧合成的超级夜景，ISP 算法处理需要好几秒。如果用户按完快门立刻把 App 退到后台甚至杀掉，以前会导致这几秒的算法中断，照片丢失或损坏。
>    从 Android 11 开始，引入了 `OfflineSession`。当 App 退到后台时，系统会接管底层的回调，在后台默默把这几秒的算法跑完并保存图库，**完美解决了后台丢图的痛点**！
> 
> 2. **不可缺少的 `clearCallingIdentity`**
>    当 C++ 底层打来电话时，代码是运行在具有超高系统权限的 Binder 线程中的。为了防止恶意的 App 借着这个系统权限环境去执行危险代码（越权攻击），Framework 必须先用 `clearCallingIdentity()` **清空系统身份，降级为普通 App 身份**，再去执行 App 开发者写的代码。这是 Android 架构安全性的典范。

---

### 📊 场景二：出炉的数据报告 (`onResultReceived`)

当一帧画面处理完成，底层的 3A 算法（自动对焦、自动曝光、自动白平衡）算出结果后，会携带包含 ISO、曝光时间等数据的 `CameraMetadataNative` 调用此方法。

```java
@Override
public void onResultReceived(CameraMetadataNative result, CaptureResultExtras resultExtras, ...) {
    long frameNumber = resultExtras.getFrameNumber();

    synchronized(mInterfaceLock) {
        // ... (省略离线会话拦截) ...
        final CaptureCallbackHolder holder = CameraDeviceImpl.this.mCaptureCallbackMap.get(requestId);
        
        // 💡 重点：判断当前拿到的是 Partial (部分) 还是 Total (最终) 结果？
        boolean isPartialResult = (resultExtras.getPartialResultCount() < mTotalPartialCount);

        Runnable resultDispatch = null;

        if (isPartialResult) {
            // 📸 路线 A：处理部分结果 (Partial Result)
            final CaptureResult resultAsCapture = new CaptureResult(getId(), result, request, resultExtras);
            
            resultDispatch = () -> {
                // 触发 App 的 onCaptureProgressed
                holder.getCallback().onCaptureProgressed(CameraDeviceImpl.this, request, resultAsCapture);
            };
            
        } else {
            // 📸 路线 B：处理最终结果 (Total Capture Result)
            // 将之前零散收集的 Partial 结果全部 pop 出来合并
            List<CaptureResult> partialResults = mFrameNumberTracker.popPartialResults(frameNumber);
            final TotalCaptureResult resultAsCapture = new TotalCaptureResult(
                    getId(), result, request, resultExtras, partialResults, ...);
                    
            resultDispatch = () -> {
                // 触发 App 的 onCaptureCompleted
                holder.getCallback().onCaptureCompleted(CameraDeviceImpl.this, request, resultAsCapture);
            };
        }

        // 依然是严密的安全保护与线程切换
        final long ident = Binder.clearCallingIdentity();
        try {
            holder.getExecutor().execute(resultDispatch);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
}
```

> **📌 源码硬核解析：极致的响应延迟优化**
> 
> 你在写 App 的时候，是否好奇过为什么要分 `onCaptureProgressed` 和 `onCaptureCompleted`？
> 
> 在底层 ISP（图像信号处理器）中，**“算出对焦数据”** 远比 **“处理完一整张 4K 图像的色彩和降噪”** 要快得多。
> Framework 层利用 `isPartialResult` 将计算快的先投递上来（组装成普通的 `CaptureResult`）。这样你的 App 就能在第一时间拿到对焦状态，并在屏幕上画出一个绿色的“对焦成功框”，让用户感觉**极其跟手、毫无延迟**。等到几百毫秒后最终图像处理完了，再把完整信息通过 `TotalCaptureResult` 投递上来。

---

## 🎯 架构师视角的总结

啃完了 `CameraDeviceImpl` 最核心的底层通信源码，我们可以深切感受到 Android Framework 工程师的良苦用心：

1. **绝对的安全隔离**
   底层与上层的交界处，永远伴随着 `synchronized` 的并发锁和 `clearCallingIdentity` 的权限搜身，绝不给 App 留下提权的漏洞。
2. **掩盖底层复杂性**
   C++ 层扔上来的只有干瘪的 `CameraMetadataNative`，Framework 层辛辛苦苦地为我们做了缓存、解包、分离（Partial/Total），最终包装成 App 开发者喜闻乐见的、极具面向对象风格的 `TotalCaptureResult` API。
3. **拥抱业务场景的演进**
   从早期的单发回调，到为高刷慢动作支持的 `hasBatchedOutputs` 批量抛出，再到为超级夜景量身定制的 `mOfflineSessionImpl`。架构在不破坏原有对外 API 的前提下，优雅地消化了硬件和业务的飞速发展。

当你下次在 `onCaptureCompleted` 回调里拿到那一长串曝光数据时，不妨在脑海里回味一下，这串数据是如何穿越 C++ 线程、绕过系统权限网、并被精心打包送到你手上的。

---

> **💬 互动环节**
> 至此，我们已经打通了 Camera Framework 层的上下行链路！关于 Android Camera，你还想了解哪方面的硬核知识？欢迎在评论区留言讨论！如果觉得本文对你有启发，别忘了点赞收藏~
```
