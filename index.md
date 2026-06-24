---
layout: home
title: Camera Learning Lab
---

# 📸 Camera Learning Lab

> **欢迎来到 Camera Learning Lab！**
>
> 这里是我从 **Android App 开发（Camera2/CameraX）** 向 **Android 系统级 Camera 开发（Framework / Native / HAL3 / ISP）** 深度进阶的系统化学习与实践记录。
> 
> 本仓库/博客旨在打破 Camera 系统的“黑盒”，建立起从上层应用、系统服务、图形内存、硬件抽象层（HAL）直至物理 Sensor/ISP 的全栈技术版图。

---

## 🗺️ 核心学习与实践版图

---

### 🟢 第一模块：App 层双轨驱动（Camera2 & CameraX）
> 巩固应用层基础，掌握 Jetpack 现代架构与传统 Camera2 的精细化控制。

- [ ] **Android CameraX 实践**
  - [ ] `Preview` 预览：生命周期绑定与 Surface 容器适配
  - [ ] `ImageCapture` 拍照：高质图片捕获与裁剪、旋转、压缩优化
  - [ ] `ImageAnalysis` 图像分析：实时帧回调与第三方算法接入
  - [ ] `Lifecycle` 绑定：解耦 Activity/Fragment 生命周期的底层原理
  - [ ] CameraX 实战常见兼容性与跳水问题记录
- [ ] **Android Camera2 核心 API 精通**
  - [ ] `CameraManager`：相机设备枚举与静态属性（`CameraCharacteristics`）读取
  - [ ] `CameraDevice`：相机硬件连接的生命周期与状态监听
  - [ ] `CameraCaptureSession`：预览、拍照、录像多路流（Surface）的创建与绑定
  - [ ] `CaptureRequest` & `CaptureResult`：控制参数下发与元数据（Metadata）接收

---

### 🟡 第二模块：跨进程通信（IPC）与图形内存血脉
> 破除多进程阻碍，理解每秒数百兆图像数据在进程间“零拷贝”传输的系统级奥秘。

- [ ] **三进程架构宏观剖析**
  - [ ] App 进程 ⇄ `cameraserver` 进程 ⇄ `android.hardware.camera.provider` 进程的边界与职责
- [ ] **跨进程通信桥梁**
  - [ ] Native Binder 机制：C++ 层的 `BpBinder` 与 `BBinder` 运作原理
  - [ ] 从 HIDL（Android 8.0+）到 AIDL-HAL（Android 12+）的接口演进
- [ ] **图形内存与共享内存机制**
  - [ ] **BufferQueue 经典生产者-消费者模型**：`dequeueBuffer`、`queueBuffer`、`acquireBuffer`、`releaseBuffer` 的状态机流转
  - [ ] **Gralloc 与 ION 内存分配**：物理连续内存与非连续内存，如何在各进程间共享内存指针实现“零拷贝”
  - [ ] **Fence 同步机制**：CPU、GPU、ISP 异步并发时的栅栏保护

---

### 🔵 第三模块：系统服务心脏 —— `libcameraservice`
> 深入 Android 系统源码，掌握相机服务的管理机制、权限拦截与硬件抢占策略。
> *核心路径：`frameworks/av/services/camera/libcameraservice/`*

- [ ] **`cameraserver` 启动与初始化**
  - [ ] `cameraserver.rc` 进程拉起解析与 Native `main()` 入口走读
- [ ] **`CameraService` 核心管理**
  - [ ] 核心类图设计：`CameraService`、`BasicClient`、`CameraDeviceClient` 的派生关系
  - [ ] **多客户端抢占机制（Conflict Resolution）**：基于 UID/PID/OOM Score 的高优先级应用强行踢掉低优先级应用的底层实现
- [ ] **`CameraProviderManager` 机制**
  - [ ] 如何动态发现底层芯片厂实现的 `ICameraProvider` 服务并映射逻辑/物理 ID

---

### 🔴 第四模块：硬件抽象层（HAL3）与规范
> 芯片厂（高通/联发科等）与手机大厂底层开发者的核心交汇点。
> *核心路径：`hardware/interfaces/camera/`*

- [ ] **HAL3 基于 Request 的管道流水线模型**
  - [ ] **In-flight Requests 机制**：硬件队列里的多帧并发与时序控制
- [ ] **AIDL/HIDL 接口剖析**
  - [ ] `configureStreams`：预览、拍照、录像流通道的底层协商与配置
  - [ ] `processCaptureRequest`：下发一帧的具体控制参数与 Buffer 载体
  - [ ] `processCaptureResult`：底层图像帧与元数据（Metadata）的异步回调
- [ ] **Stream 分类**：Output 流、Input 重新处理流（用于 Reprocess）、双向流

---

### 🟣 第五模块：四大核心工作流（源码级时序图）
> 贯穿 Java -> JNI -> Native Client -> cameraserver -> HAL3 的完整链路分析。

- [ ] **1. Open Camera 流程**（应用打开相机到硬件就绪）
- [ ] **2. Configure Streams 流程**（配置通道与 Gralloc 批量分配 Buffer）
- [ ] **3. Start Preview / Capture 流程**（循环下发 Request 与数据帧回调）
- [ ] **4. Close Camera 流程**（资源释放、流注销与 Binder 断开）

---

### 🟤 第六模块：物理 Sensor、ISP 管道与 3A 算法
> 揭秘光线如何一步步变成像素。理解物理硬件，与算法、驱动工程师无缝沟通。

- [ ] **物理 Sensor 与数据传输**
  - [ ] 光电效应、拜耳阵列（Bayer Pattern, RGBG）、四合一像素原理
  - [ ] **MIPI CSI-2 协议**（传输图像 RAW 数据）与 **I2C/I3C 协议**（下发控制寄存器）
- [ ] **ISP (Image Signal Processor) 管道魔术**
  - [ ] BLC（去黑电平） ➔ LSC（镜头阴影校正） ➔ Demosaic（去马赛克插值） ➔ AWBG（白平衡增益） ➔ CCM（色彩矩阵校正） ➔ Gamma（伽马校正） ➔ Denoise（降噪）
- [ ] **3A 算法控制**
  - [ ] **AE（自动曝光）**：快门、光圈、ISO 与直方图亮度收敛
  - [ ] **AF（自动对焦）**：Contrast AF（反差）、PDAF（相位）、TOF/Laser（激光对焦）
  - [ ] **AWB（自动白平衡）**：灰色世界假设、色温计算

---

### 🧡 第七模块：高级 Camera 特性与前沿技术
> 进阶高级系统开发的核心加分项，涉及主流旗舰机的核心相机玩法。

- [ ] **ZSL (Zero Shutter Lag, 零延时拍照)**
  - [ ] 预览后台 YUV/RAW 环形缓冲区（RingBuffer）设计与时间戳无缝匹配
- [ ] **Reprocessing (重处理技术)**
  - [ ] RAW/YUV 输入流重回 ISP 管道，运行 HDR、超级夜景等后处理算法
- [ ] **Multi-Camera (多摄架构)**
  - [ ] 逻辑 Camera 与物理 Camera 的映射关系
  - [ ] 变焦平滑切换（Zoom Transition）与双摄帧同步
  - [ ] 人像虚化（Bokeh）底层的双目测距与深度图（Depth Map）生成

---

### 🎨 第八模块：图像格式、编解码与数据解析
> 玩转底层的像素矩阵，掌握高效的数据转换方法。

- [ ] **核心图像格式剖析**
  - [ ] **YUV 格式深度解析**：YUV420P、YUV420SP、NV21、NV12 的内存排布差异
  - [ ] **JPEG & Bitmap**：编解码原理、内存占用分析与高效互转
- [ ] **图像操作实战**
  - [ ] 使用 **libyuv** 库在 Native 层对 YUV 帧进行极速旋转、裁剪、缩放与格式转换
  - [ ] `ImageReader` 原始数据字节流（Byte Buffer）的手动步长（Stride）解析与避坑

---

### 🛠️ 第九模块：Bug 排查、系统调优与实战武器库
> 解决现实中的疑难杂症，从容应对大厂性能指标要求。

- [ ] **经典应用层 Bug 攻克**
  - [ ] 预览变形/拉伸（AspectRatio 比例不适配问题）
  - [ ] 预览黑屏（Surface 生命周期、线程死锁、底层 Device 挂死）
  - [ ] 图片旋转错误（Sensor 角度 Exif Orientation 补偿计算）
  - [ ] 多机型兼容性适配与防崩溃策略
- [ ] **底层核心调试工具**
  - [ ] `adb shell dumpsys media.camera` 实时相机状态抓取
  - [ ] **Perfetto / Systrace 深度使用**：
    - 首帧出图慢（Launch Latency）耗时瓶颈排查
    - 预览掉帧、卡顿（Jank）的瓶颈定位（App、cameraserver 还是 HAL 驱动处理慢）
  - [ ] C++ 内存分析：利用 ASan (AddressSanitizer) 查找 Native 内存越界与泄漏
- [ ] **系统级测试**：Camera CTS (兼容性测试) / ITS (图像传输测试) 运行与问题修复

---

## 📅 学习进度追踪

- [x] Camera 基础 - 相机预览与拍照流程
- [ ] CameraX 应用开发与实践记录
- [ ] Camera2 核心类分析
- [ ] 深入剖析 BufferQueue 零拷贝机制
- [ ] 走读 openCamera 完整跨进程调用链
- [ ] 掌握 ISP 管道与 3A 控制机制
- [ ] ZSL 与 Reprocess 实战笔记
- [ ] Systrace 性能分析实战

---

> *"The best way to predict the future is to invent it."*
> 欢迎关注我的博客，一起在这个 Camera Learning Lab 里见证从应用到系统底层的蜕变！
