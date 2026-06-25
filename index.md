---
layout: home
title: "Camera Learning Lab - Yang Bolin"
---

# Camera Learning Lab

👋 你好，我是 **Yang Bolin**。专注于车载智能视界开发，深耕 **Android Camera FW 框架层、HAL3 标准规范以及底层核心驱动接口**。

致力于打通图像采集（TP2856 / TP9951）至 NPU 神经网络（RK3562）的极速零拷贝（Zero-Copy）数据管道。

---

## 📝 最新文章 / Tech Posts

{% for post in site.posts %}
### 📄 [{{ post.title }}]({{ post.url | relative_url }})
*📅 发布于 {{ post.date | date: "%Y-%m-%d" }}*

{% if post.excerpt %}
{{ post.excerpt | strip_html | strip_newlines | truncate: 150 }}
{% else %}
点击深入探索此篇底层 Camera 技术细节...
{% endif %}

---
{% endfor %}

## 🛠️ 我的研究体系 / Tech Stack

*   **应用与 FW 层**：Camera2/CameraX 控制栈、`cameraserver` 核心管理、JNI 数据强转与脱壳、多应用权限抢占策略。
*   **HAL & 驱动层**：Camera HAL3 (AIDL/HIDL)、`BufferQueue` 状态机流转、V4L2 驱动机制与 `ioctl`、MIPI Virtual Channel。
*   **底层控制与硬件**：ION/Gralloc 内存共享、TP2856/TP9951 解码芯片寄存器、`dma-buf` 零拷贝技术、NPU 内存直接映射。
