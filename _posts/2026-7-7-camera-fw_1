```markdown
# 📚 Android Camera Framework 源码精读笔记 (第二部分)

> **📝 导读**
> 本节笔记继续深入 `frameworks/base/core/java/android/hardware/camera2/impl/CameraDeviceImpl.java`。
> 我们将聚焦于内部类 `CameraDeviceCallbacks`。它的身份是 **App 留在底层的“客服接线员”**，所有来自底层 C++ (`cameraserver` 进程) 的通知（如：出图了、报错了、曝光了），都要由这个类来接收，并安全地转发给你的 App。

---

## 📍 核心前提：认识接线员与离线会话

在看具体的方法前，我们必须先理解两个极其关键的背景知识：

1. **Binder 线程池**：底层 C++ 呼叫这里的方法时，代码是运行在匿名的 Binder 线程里的，**绝对不能**在这里直接更新 UI 或处理耗时操作，必须切换回 App 指定的线程。
2. **`mOfflineSessionImpl` (离线处理机制)**：
   - **这是什么？** 这是 Android 11 (API 30) 引入的超级杀手锏功能。
   - **为什么要有它？** 现在的手机拍照动辄 1 亿像素，或者超级夜景，底层算法处理需要好几秒。如果用户按完快门，立刻把 App 退到后台或者杀掉，以前会导致这张照片丢失/损坏。
   - **它的作用**：有了 `OfflineSession`，如果 App 处于半退出的离线切换状态，底层的回调会被它**拦截并接管**，在后台默默把这几秒的算法跑完并保存，完美解决后台丢图问题！

---

## 👣 源码逐段拆解

### 🍰 第一段：基础信息转移 —— `onDeviceError` 与 `onDeviceIdle`

最基础的回调，收到消息后直接转发给外部大老板（`CameraDeviceImpl`）去处理。

```java
public class CameraDeviceCallbacks extends ICameraDeviceCallbacks.Stub {

    @Override
    public IBinder asBinder() {
        return this; // 告诉系统：我就是一个支持跨进程通信的 Binder 对象
    }

    @Override
    public void onDeviceError(final int errorCode, CaptureResultExtras resultExtras) {
        // 底层说：相机坏了/被抢占了！
        // 接线员自己不处理，直接转交给外部类去处理
        CameraDeviceImpl.this.onDeviceError(errorCode, resultExtras);
    }

    @Override
    public void onDeviceIdle() {
        // 底层说：队列里所有的请求都拍完了，我现在闲下来了
        CameraDeviceImpl.this.onDeviceIdle();
    }
}
```

---

### 🍰 第二段：快门按下的瞬间 —— `onCaptureStarted`

这是最重要的回调之一！当感光元件 (Sensor) 真正开始收集光子（曝光）的那一瞬间，底层会立刻调用此方法。你的 App 通常在这里播放“咔嚓”的快门声。

```java
@Override
public void onCaptureStarted(final CaptureResultExtras resultExtras, final long timestamp) {
    // 1. 从底层的快递包裹 (resultExtras) 拆出帧号、请求ID等
    int requestId = resultExtras.getRequestId();           
    final long frameNumber = resultExtras.getFrameNumber(); 

    synchronized(mInterfaceLock) { 
        // 🚨 必须加锁！因为底层可能有多条 C++ 线程同时打电话过来
        
        if (mRemoteDevice == null) return; // 相机已关，直接挂断电话

        // 🌟 核心拦截：离线会话 (Offline Session) 的介入
        // 如果当前处于离线切换状态，这个回调就不给原 App 发了，交给 OfflineSession 处理
        if (mOfflineSessionImpl != null) {
            mOfflineSessionImpl.getCallbacks().onCaptureStarted(resultExtras, timestamp);
            return;
        }

        // 2. 找到当初是谁（哪个 App 的 Request）下的这个订单
        final CaptureCallbackHolder holder = CameraDeviceImpl.this.mCaptureCallbackMap.get(requestId);
        if (holder == null) return; 

        // 3. 🚨 重点操作：身份切换与线程抛转！
        final long ident = Binder.clearCallingIdentity(); // 脱下底层系统的权限马甲
        try {
            // 切回 App 当初传入的 Executor 线程
            holder.getExecutor().execute(new Runnable() {
                @Override
                public void run() {
                    // 4. 判断是否是“批量输出” (BatchedOutputs)
                    // (多用于 120/240fps 慢动作录像，底层为了省事，把多帧打包一次性发过来)
                    if (holder.hasBatchedOutputs()) {
                        for (int i = 0; i < holder.getRequestCount(); i++) {
                            holder.getCallback().onCaptureStarted(...); // 循环拆包回调
                        }
                    } else {
                        // 正常情况，回调给 App 开发者写的 onCaptureStarted
                        holder.getCallback().onCaptureStarted(
                            CameraDeviceImpl.this, request, timestamp, frameNumber);
                    }
                }
            });
        } finally {
            Binder.restoreCallingIdentity(ident); // 穿回系统马甲
        }
    }
}
```

**👨‍🏫 老师敲黑板**：`Binder.clearCallingIdentity()`
底层打来电话时，带着的是**系统进程 (System)** 的超高权限。如果要执行你 App 里写的代码，必须先用这行代码“清空系统身份，恢复成本地普通 App 身份”。这是 Android 防止恶意 App 借机提权搞破坏的核心安全机制。

---

### 🍰 第三段：出炉的数据报告 —— `onResultReceived`

当一帧画面处理完成，底层的 3A 算法（自动对焦、自动曝光等）算出所有元数据（Metadata，比如 ISO、曝光时间）后，调用这里返回数据。

```java
@Override
public void onResultReceived(CameraMetadataNative result, CaptureResultExtras resultExtras, ...) {
    long frameNumber = resultExtras.getFrameNumber();

    synchronized(mInterfaceLock) {
        // ... (离线会话拦截：mOfflineSessionImpl != null 判断同上) ...

        final CaptureCallbackHolder holder = CameraDeviceImpl.this.mCaptureCallbackMap.get(requestId);
        
        // 💡 重点判断：这是“部分结果”还是“最终结果”？
        boolean isPartialResult = (resultExtras.getPartialResultCount() < mTotalPartialCount);

        Runnable resultDispatch = null;
        CaptureResult finalResult;

        if (isPartialResult) {
            // 📸 情况A：部分结果 (Partial Result)
            // 为什么需要部分结果？因为 3A 算法中“对焦”算得特别快。
            // 提前把对焦结果发回来，App 就能立刻在 UI 画上绿色的“对焦成功框”，让用户感觉毫无延迟！
            final CaptureResult resultAsCapture = new CaptureResult(getId(), result, request, resultExtras);
            
            resultDispatch = new Runnable() {
                public void run() {
                    // 回调到 App 的 onCaptureProgressed (进行中)
                    holder.getCallback().onCaptureProgressed(CameraDeviceImpl.this, request, resultAsCapture);
                }
            };
            
        } else {
            // 📸 情况B：最终结果 (Total Capture Result)
            // 图像的全部数据都齐了！把之前零散的所有 Partial 结果取出来合并
            List<CaptureResult> partialResults = mFrameNumberTracker.popPartialResults(frameNumber);
            
            final TotalCaptureResult resultAsCapture = new TotalCaptureResult(
                    getId(), result, request, resultExtras, partialResults, ...);
                    
            resultDispatch = new Runnable() {
                public void run() {
                    // 回调到 App 的 onCaptureCompleted (已完成)
                    holder.getCallback().onCaptureCompleted(CameraDeviceImpl.this, request, resultAsCapture);
                }
            };
        }

        // 依然需要 clearCallingIdentity 保驾护航，跨线程去执行
        final long ident = Binder.clearCallingIdentity();
        try {
            holder.getExecutor().execute(resultDispatch);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
}
```

**👨‍🏫 老师敲黑板**：`CaptureResult` vs `TotalCaptureResult`
这段代码完美解释了 App API 的区别：`onCaptureProgressed` 丢给你的是 `CaptureResult`（信息不全），而 `onCaptureCompleted` 丢给你的是 `TotalCaptureResult`（包含了所有的部分结果，是照片的完整 EXIF 级属性集合）。

---

## 🎓 本节小结

通过拆解这段真实的、包含 `mOfflineSessionImpl` 的高版本 Android 源码，我们可以清晰地看到 Framework 层的精妙设计：

1. **新老交替的桥梁**：通过 `mOfflineSessionImpl`，系统可以在不惊动上层业务逻辑的情况下，悄悄把耗时的拍照任务转移到后台继续处理，体现了极好的扩展性。
2. **严丝合缝的安全网**：在从底层跨越到应用层的前夕，必须加 `synchronized` 锁防止线程冲突，并且必须经历 `clearCallingIdentity` 的“搜身”，防止 App 越权。
3. **分阶段的数据投递**：利用 `isPartialResult` 将计算快的（对焦）和计算慢的（成像）分开投递，极致压榨系统的响应速度。
```

老师说明：这第二份笔记已经包含了你提供的那一长串源码中最核心、最精髓的部分，包括了 `mOfflineSessionImpl`、线程安全、身份清理以及 Partial/Total 的区分。你可以细细品味一下这些底层架构师的设计思路。如果这部分也看懂了，随时告诉我，我们可以继续探讨接下来的流程（比如底层 C++ 或 画面渲染机制）！
