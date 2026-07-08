```markdown
# 📚 Android 16+ Camera Framework 源码精读：CameraManager 详解

> **📝 课前导读**
> 本节笔记基于最新 Android (16+) AOSP main 分支源码。
> `CameraManager` 是我们在 App 中调用最频繁的类。但在现代 Android 架构中，`CameraManager` 已经退居幕后，变成了一个“皮包公司”和“门面担当”，真正的核心逻辑被隐藏在了一个名为 `CameraManagerGlobal` 的单例对象中。

---

## 📍 核心架构：门面与大管家

在看代码之前，我们要建立现代 Android Camera 框架的心智模型：

*   **`CameraManager` (大堂经理)**：每个 Context（比如你的 Activity）向系统获取的其实都是一个**全新的 `CameraManager` 实例**。它只负责维护和当前 App 上下文相关的状态。
*   **`CameraManagerGlobal` (全球调度中心)**：这是一个**单例（Singleton）**。不管你创建了多少个 `CameraManager`，它们在底层都共享同一个 `CameraManagerGlobal`。它负责和底层的 C++ `cameraserver` 保持长连接（Binder 通信）。

---

## 👣 源码逐段拆解

### 🍰 第一段：获取可用相机列表 `getCameraIdList()`

你肯定写过 `cameraManager.getCameraIdList()`。我们来看看它到底怎么拿到数据的。

#### 💻 Android 16+ 源码：
```java
// 源码位置：CameraManager.java

public @NonNull String[] getCameraIdList() throws CameraAccessException {
    // 💡 这里的 CameraManagerGlobal.get() 就是去获取全局调度中心
    return CameraManagerGlobal.get().getCameraIdList();
}

// -------------------------------------------------------------
// 👇 下面是跳转到 CameraManagerGlobal.java 内部的实现
// -------------------------------------------------------------
public String[] getCameraIdList() {
    String[] cameraIds = null;
    synchronized (mLock) {
        try {
            // 1. 尝试连接底层的 C++ CameraService
            connectCameraServiceLocked();
            
            // 2. 跨进程调用底层接口，获取所有的相机 ID
            // 注意：底层返回的不仅有你的手机镜头，可能还有外接 USB 镜头、虚拟镜头等
            cameraIds = mCameraService.getCameraIdList();
            
            // 3. 🚦 现代 Android 新特性：过滤与降级！
            // 为什么需要 filter？
            // 假设你在用一个很老的 App（targetSdkVersion 比较低），
            // 底层查到了一个只能输出 10bit HDR 的高级镜头，老 App 根本不支持。
            // 这里的 filter 就会把这个高级镜头从列表中“隐藏”掉，防止 App 崩溃。
            cameraIds = filterCameraIds(cameraIds);
            
        } catch (RemoteException e) {
            // 底层服务死掉了
        }
    }
    return cameraIds != null ? cameraIds : new String[0];
}
```

**👨‍🏫 老师讲解**：
你会发现，`CameraManager` 自己啥也不干，直接抛给 `CameraManagerGlobal`。而在 Android 16+ 中，最大的特点就是加入了严格的**设备过滤机制 (`filterCameraIds`)**。这体现了系统对老旧 App 极强的向后兼容保护。

---

### 🍰 第二段：最核心的入口 `openCamera()`

这个方法我们在之前的笔记中提过，但在 Android 16+ 中，它对线程和上下文的处理更加严谨。

#### 💻 Android 16+ 源码：
```java
// 源码位置：CameraManager.java

@RequiresPermission(android.Manifest.permission.CAMERA)
public void openCamera(@NonNull String cameraId,
        @NonNull @CallbackExecutor Executor executor,
        @NonNull final CameraDevice.StateCallback callback)
        throws CameraAccessException {
        
    // 1. 基础校验（防止你乱传 null）
    if (cameraId == null) {
        throw new IllegalArgumentException("cameraId was null");
    }
    if (callback == null) {
        throw new IllegalArgumentException("callback was null");
    }

    // 2. 获取 App 的身份信息 (很重要！)
    // context.getOpPackageName() 获取你的包名
    // context.getAttributionTag() 获取特性标签 (Android 12+ 引入，用于精细化隐私审计)
    String opPackageName = mContext.getOpPackageName();
    String attributionTag = mContext.getAttributionTag();
    int uid = mContext.getApplicationInfo().uid; // 你的 App 唯一 ID

    // 3. 准备进程优先级信息
    int oomScoreOffset = 0; // OOM 评分偏移量，系统可以根据这个决定在内存不足时先杀谁

    // 4. 调用内部的异步方法去执行
    openCameraDeviceUserAsync(cameraId, callback, executor, uid, 
                              opPackageName, attributionTag, oomScoreOffset);
}
```

**👨‍🏫 老师讲解**：
与旧版本相比，现代 Android 强推 `Executor` 而非 `Handler`。同时，引入了 `attributionTag`（归因标签）。
**举个例子**：如果你的 App 既有普通的“拍照功能”，又有一个“后台扫码插件”。通过传递不同的 `attributionTag`，Android 系统的“隐私仪表板（Privacy Dashboard）”就能精准地告诉用户：“你的 App 是为了**后台扫码**才调用的相机，而不是在偷偷拍照”。

---

### 🍰 第三段：惊险的跨进程跃迁 `openCameraDeviceUserAsync()`

真正干活的地方来了！这里是 Java 框架层与 C++ 底层服务的**交汇点**。

#### 💻 Android 16+ 源码：
```java
// 源码位置：CameraManager.java

private CameraDevice openCameraDeviceUserAsync(String cameraId,
        CameraDevice.StateCallback callback, Executor executor, int uid, 
        String opPackageName, String attributionTag, int oomScoreOffset) 
        throws CameraAccessException {

    // 1. 拿到相机的只读配置表 (分辨率支持、是否支持夜景等)
    CameraCharacteristics characteristics = getCameraCharacteristics(cameraId);
    
    // 2. 🚦 现代 Android 新特性：检查折叠屏的设备状态 (Device State)
    // 折叠屏手机在展开和闭合时，同一个相机的物理位置可能不同！
    // mDeviceId 保存了当前是内屏还是外屏
    int deviceId = mDeviceId;

    // 3. 实例化干活的打工仔 (CameraDeviceImpl)
    CameraDeviceImpl deviceImpl = new CameraDeviceImpl(
            cameraId, callback, executor, characteristics, 
            mContext.getApplicationInfo().targetSdkVersion, 
            mContext);

    // 4. 获取用来接收底层 C++ 电话的“传声筒”
    ICameraDeviceCallbacks callbacks = deviceImpl.getCallbacks();

    try {
        // 5. 再次向 Global 大管家要底层的跨进程服务代理
        ICameraService cameraService = CameraManagerGlobal.get().getCameraService();
        if (cameraService == null) {
            throw new CameraAccessException(CameraAccessException.CAMERA_DISCONNECTED, ...);
        }

        // 6. 🔥 真正的跨进程 IPC 调用来了！
        // 带着所有的身家性命（包名、UID、权限标签）请求 C++ 底层开机！
        cameraService.connectDevice(
                callbacks,           // 底层开机后的回调通道
                cameraId,            // 要开哪个相机？
                opPackageName,       // 你的包名 (隐私审查用)
                attributionTag,      // 你的特性标签 (隐私审计用)
                uid,                 // 你的进程 UID
                oomScoreOffset,      // 内存优先级
                /*out*/ deviceImpl.getRemoteDevice()); // 💡 输出参数：把底层的控制句柄塞回给 Java 层
                
    } catch (ServiceSpecificException e) {
        // 7. 处理底层 C++ 抛回来的专属异常
        // 比如：ERROR_CAMERA_IN_USE (被微信占用了)，ERROR_MAX_CAMERAS_IN_USE (同时开太多了)
        throwAsPublicException(e);
    } catch (RemoteException e) {
        // 8. 处理通信异常 (比如 C++ 进程崩溃了)
        throw new CameraAccessException(CameraAccessException.CAMERA_DISCONNECTED, ...);
    }

    // 9. 返回打工仔实例给你的 App
    return deviceImpl;
}
```

**👨‍🏫 老师讲解**：
仔细看注释 第 2 步 和 第 6 步！
*   **第 2 步的折叠屏支持**：随着折叠屏的大火，Android 源码里随处可见 `deviceId` 和 `deviceState` 的判断。这就解释了为什么当折叠屏合上时，系统能自动帮你切换或纠正摄像头参数。
*   **第 6 步的异常捕获**：在较老的 Android 版本中，底层异常经常导致应用直接闪退。但在 Android 16 中，系统通过 `ServiceSpecificException` 精准捕获了 C++ 层定义的错误码（比如别的 App 正在独占相机），然后通过 `throwAsPublicException(e)` 翻译成开发者能看懂的 `CameraAccessException`，非常优雅！

---

### 🍰 第四段：折叠屏/并发功能的注册监听 (现代 Android 独有)

除了打开相机，`CameraManager` 在现代 Android 中还肩负着监听物理设备变化的重任。

#### 💻 Android 16+ 源码：
```java
// 源码位置：CameraManager.java

// 注册相机可用性状态的回调
public void registerAvailabilityCallback(@NonNull @CallbackExecutor Executor executor,
        @NonNull AvailabilityCallback callback) {
        
    // 💡 你发现了吗？又是直接甩锅给 Global！
    CameraManagerGlobal.get().registerAvailabilityCallback(callback, executor);
}

// -------------------------------------------------------------
// 👇 在 CameraManagerGlobal 内部
// -------------------------------------------------------------
public void registerAvailabilityCallback(...) {
    synchronized (mLock) {
        // 1. 如果底层的 C++ 服务还没连接，赶紧连一下
        connectCameraServiceLocked();
        
        // 2. 把 App 传入的 callback 存进一个 Map 里
        mCallbackMap.put(callback, executor);
        
        // 3. 立即触发一次回调，告诉 App 当前相机的状态
        // 💡 为什么遍历 mDeviceStatusMap？
        // 因为 C++ 底层只要相机状态一变，就会更新 mDeviceStatusMap。
        // 这就是为什么微信能知道你什么时候合上了折叠屏的相机！
        for (Map.Entry<String, Integer> status : mDeviceStatusMap.entrySet()) {
            String cameraId = status.getKey();
            int statusValue = status.getValue();
            
            if (statusValue == ICameraServiceListener.STATUS_PRESENT) {
                // 相机可用！通过 executor 抛回给 App
                executor.execute(() -> callback.onCameraAvailable(cameraId));
            } else {
                // 相机不可用！
                executor.execute(() -> callback.onCameraUnavailable(cameraId));
            }
        }
    }
}
```

**👨‍🏫 老师讲解**：
如果你的 App 想做一个“智能相机”，比如当用户插上外接 USB 摄像头时，或者别的 App 退出了相机时，你的 App 能够立刻自动捕捉到并弹窗提示。靠的就是这个机制！
`CameraManagerGlobal` 在底层注册了一个 `ICameraServiceListener`，它像雷达一样监听全局的相机拔插、占用事件，并分发给所有注册了 `AvailabilityCallback` 的 App。

---

## 🎓 班会总结：Android 16 级 Framework 工程师的视野

通过拆解 `CameraManager.java`，作为小白的你现在应该具备了以下高级视野：

1. **单例代理模式的巅峰 (`CameraManagerGlobal`)**：
   为了避免每个 `Context` 都去创建一个极度消耗资源的跨进程 Binder 连接，Google 采用了一个全局单例来统管所有的 IPC 通信，`CameraManager` 只是提供给 App 的 API 壳子。
2. **极度细化的隐私保护 (`attributionTag`)**：
   了解现代系统是如何追踪哪一行代码、哪一个功能在请求相机的，不再是一笔糊涂账。
3. **向物理硬件妥协的架构设计 (`DeviceState`)**：
   源码中处处可见折叠屏、多屏设备带来的影响。系统要在底层默默屏蔽掉这些物理差异，把干净统一的 `CameraDevice` 接口交给你。

> 同学，把这三份笔记（[宏观篇]、[接线员篇]、[前台大堂经理篇]）结合起来看，Android Camera 从你 App 敲下第一行代码，一直到底层 C++ 通电开机的**纵向高速公路**，已经完全向你敞开了！
```
