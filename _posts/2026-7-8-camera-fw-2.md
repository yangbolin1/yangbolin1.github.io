***

# 深入底层：Android Camera2 Framework 源码硬核解析 —— openCamera 背后发生了什么？

## 📝 前言

作为 Android 应用开发者，你一定对 `Camera2` API 不陌生。我们轻车熟路地调用 `cameraManager.openCamera()`，在回调中拿到 `CameraDevice`，然后开启预览。

但你是否思考过，这短短的几行 Java 代码背后，Android 系统究竟做了什么？
- 我们的请求是如何跨越 Java 的边界到达 C++ 底层的？
- 现代 Android（14/15/16）在折叠屏和多摄并发上，对框架做了哪些重构？
- 底层 C++ 是如何绕过主线程，安全地把数据回调给我们的 App 的？

今天，我们将一头扎进 Android AOSP 源码树的 `frameworks/base/core/java/android/hardware/camera2/` 目录，以最新的 **Android 16+** 架构为视角，扒开 `CameraManager` 的外衣，一探究竟！

---

## 🏗️ 核心架构：重新认识 CameraManager

在深入源码之前，我们需要打破一个常规认知：**`CameraManager` 并不真正“管理”相机。**

在现代 Android 架构中，真正的设计模式是**“门面与大管家”**：

1. **`CameraManager` (前台门面)**：你的 Activity 通过 `getSystemService` 获取的只是一个轻量级的包装类，它负责处理当前 Context 的状态、权限校验。
2. **`CameraManagerGlobal` (幕后管家)**：这是一个**全局单例**。不论你的 App 创建了多少个 `CameraManager`，所有跨进程（IPC）与底层 C++ (`cameraserver`) 的通信，全部由它统一接管。

```text
[你的 App 进程]
  ├── CameraManager (实例A) ─┐
  ├── CameraManager (实例B) ─┼─▶ CameraManagerGlobal (全局单例)
                                      │
                                      ▼ Binder IPC 通信
===================================================================
[系统 cameraserver 进程 (C++)]
  └── CameraService
```

建立好这个心智模型后，我们开始追踪 `openCamera` 的源码链路。

---

## 👣 链路追踪：一步步拆解 openCamera

### Step 1：前台接待 —— 参数校验与隐私追踪

我们在 App 中调用的公开 API，其实是一个只做“前置检查”的代理方法。

```java
// 源码位置：CameraManager.java

@RequiresPermission(android.Manifest.permission.CAMERA)
public void openCamera(@NonNull String cameraId,
        @NonNull @CallbackExecutor Executor executor,
        @NonNull final CameraDevice.StateCallback callback)
        throws CameraAccessException {
        
    // 1. 基础防呆校验
    if (cameraId == null || callback == null) {
        throw new IllegalArgumentException("cameraId or callback was null");
    }

    // 2. 💡 现代 Android 隐私审计特性
    String opPackageName = mContext.getOpPackageName();
    String attributionTag = mContext.getAttributionTag(); // 特性标签
    int uid = mContext.getApplicationInfo().uid; 

    // 3. 进程优先级设置
    int oomScoreOffset = 0; 

    // 4. 转入幕后异步处理
    openCameraDeviceUserAsync(cameraId, callback, executor, uid, 
                              opPackageName, attributionTag, oomScoreOffset);
}
```

> **📌 源码亮点解析：`attributionTag` (归因标签)**
> 从 Android 12 开始，系统对隐私控制极其严格。如果你 App 内不仅有主干的“拍照”功能，还有“后台静默扫码”模块。通过传递不同的 `attributionTag`，Android 的隐私仪表板就能精准记录并告诉用户：你的 App 到底是为了哪个具体业务调用的摄像头。

---

### Step 2：幕后组装 —— 准备跨进程跃迁

进入 `openCameraDeviceUserAsync` 后，真正干活的打工仔实例开始被创建，并准备发起向底层的冲锋。

```java
// 源码位置：CameraManager.java

private CameraDevice openCameraDeviceUserAsync(...) throws CameraAccessException {

    // 1. 获取相机的只读配置表 (Metadata)
    CameraCharacteristics characteristics = getCameraCharacteristics(cameraId);
    
    // 2. 💡 折叠屏支持：记录当前物理设备状态
    int deviceId = mDeviceId;

    // 3. 创建真正的相机实例：CameraDeviceImpl
    CameraDeviceImpl deviceImpl = new CameraDeviceImpl(
            cameraId, callback, executor, characteristics, ...);

    // 4. 关键：拿到接收底层回调的 Binder 服务端 (传声筒)
    ICameraDeviceCallbacks callbacks = deviceImpl.getCallbacks();

    try {
        // 5. 获取全局单例，拿到 C++ 层的 Binder 代理对象
        ICameraService cameraService = CameraManagerGlobal.get().getCameraService();

        // 6. 🔥 真正的 IPC 跨进程调用：请求底层通电开机！
        cameraService.connectDevice(
                callbacks,           // 把“传声筒”递给底层
                cameraId,            // 要开哪个相机？
                opPackageName,       // App 包名
                attributionTag,      // 隐私归因标签
                uid,                 // App UID
                oomScoreOffset,      // 内存 OOM 优先级
                /*out*/ deviceImpl.getRemoteDevice()); // 拿到底层的控制句柄
                
    } catch (ServiceSpecificException e) {
        // 将 C++ 定义的错误码翻译为 Java 层异常
        throwAsPublicException(e);
    } 

    return deviceImpl;
}
```

> **📌 源码亮点解析：向折叠屏妥协的架构**
> 现代源码中到处可见 `deviceId` 的身影。当折叠屏手机展开或闭合时，同一个逻辑 `cameraId` 对应的物理摄像头可能会发生切换。Framework 层在这里默默记录了 DeviceState，为后续纠正物理传感器的参数做准备。

---

### Step 3：底层接线员 —— CameraDeviceCallbacks 

在 Step 2 中，我们把 `callbacks` 传给了底层 C++ 服务。当底层硬件成功开机、或者传感器开始曝光时，它会通过 Binder 跨进程打回电话。

接电话的就是 `CameraDeviceImpl` 内部的 `CameraDeviceCallbacks`。这是极其精彩的一段代码，展示了极致的**线程安全与权限隔离**。

```java
// 源码位置：CameraDeviceImpl.java 的内部类

public class CameraDeviceCallbacks extends ICameraDeviceCallbacks.Stub {

    @Override
    public void onCaptureStarted(final CaptureResultExtras resultExtras, final long timestamp) {
        int requestId = resultExtras.getRequestId();           

        synchronized(mInterfaceLock) { 
            // 🚨 必须加锁：底层可能有多个 C++ 线程并发回调
            
            // 💡 Android 11+ 离线会话拦截
            if (mOfflineSessionImpl != null) {
                mOfflineSessionImpl.getCallbacks().onCaptureStarted(resultExtras, timestamp);
                return;
            }

            final CaptureCallbackHolder holder = CameraDeviceImpl.this.mCaptureCallbackMap.get(requestId);
            if (holder == null) return; 

            // 🔥 核心安全机制：脱下系统马甲
            final long ident = Binder.clearCallingIdentity(); 
            try {
                // 切换回 App 开发者最初传入的 Executor 线程
                holder.getExecutor().execute(() -> {
                    // 回调给 App 层的代码
                    holder.getCallback().onCaptureStarted(
                        CameraDeviceImpl.this, request, timestamp, frameNumber);
                });
            } finally {
                // 穿回系统马甲
                Binder.restoreCallingIdentity(ident); 
            }
        }
    }
}
```

> **📌 源码亮点解析：**
> 1. **权限隔离 (`clearCallingIdentity`)**：底层 C++ 回调时，带有极高的系统级 Binder 身份。为了防止恶意 App 利用这个回调环境进行提权攻击，系统必须先用 `clearCallingIdentity` 清理系统身份，降级为普通 App 身份，再执行 `Executor` 中的代码。
> 2. **`OfflineSession` 机制**：如果 App 退到了后台，但高像素的图片还在处理中，`mOfflineSessionImpl` 会在此拦截回调，接管后续的算法处理，完美防止后台丢图。

---

## 🎯 总结与启发

通过这次对 `frameworks/base` 源码的“开膛破肚”，我们可以看到 Android 架构师在设计 Framework 时的核心思想：

1. **极致的资源管理 (Singleton Binder IPC)**
   为了避免每个 Context 都创建昂贵的跨进程连接，采用 `CameraManagerGlobal` 单例接管所有底层通信，应用层的 API 仅仅是个轻量级的壳。
2. **严丝合缝的安全网**
   从调用 `openCamera` 时的 `uid` 和 `attributionTag` 审查，到回调时的 `clearCallingIdentity` 身份降级，处处体现了系统对权限安全的警惕。
3. **拥抱硬件演进**
   不管是处理 120fps 高刷带来的 `BatchedOutputs` 批量回调，还是兼容折叠屏状态的 `DeviceState` 记录，Framework 层始终在做“脏活累活”，向我们上层提供干净统一的 API。

理解了这些，当你下次在项目中敲出 `cameraManager.openCamera(...)` 时，相信你的脑海中定能浮现出一条贯穿应用、Framework、直至 C++ 底层的清晰数据流！

---

> **💬 互动环节**
> 关于 Android Camera 底层原理，你还想了解哪一部分？是 *GraphicBuffer 零拷贝渲染机制*，还是 *HAL 层的 3A 算法流转*？欢迎在评论区留言讨论！如果觉得本文对你有帮助，不妨点赞收藏支持一下~
