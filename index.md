---
layout: default
title: "Camera Learning Lab - Yang Bolin"
---

<div align="center">
  <h1>📸 Camera Learning Lab</h1>
  <p><i>专注于车载智能视界开发 · 探索极致的图像数据流</i></p>
</div>

<br>

> 👋 **你好，我是 Yang Bolin。**
> 
> 致力于打通从camera图像采集至 NPU 神经网络（RK3562）的极速**零拷贝（Zero-Copy）**数据管道。深耕 Android Camera FW 框架层、HAL3 标准规范以及底层核心驱动接口。

---

## 🛠️ 核心研究体系 / Tech Stack

为了更清晰地展示，我将技术栈进行了结构化分类：

| 技术领域 | 核心技能点与深入方向 |
| :--- | :--- |
| 🚀 **应用与 FW 层** | `Camera2 / CameraX` 控制栈深度定制<br>`cameraserver` 核心服务与跨进程通信管理<br>`JNI` 数据强转与脱壳处理<br>多应用并发下的 Camera 权限抢占策略 |
| ⚙️ **HAL & 驱动层** | `Camera HAL3 (AIDL/HIDL)` 接口实现与优化<br>`BufferQueue` 状态机复杂流转机制<br>`V4L2` 驱动底层机制与自定义 `ioctl`<br>`MIPI Virtual Channel` 数据分离 |
| 🧲 **底层控制与硬件** | `ION / Gralloc` 跨模块内存共享机制<br>`dma-buf` 零拷贝技术全链路打通<br> 视频解码芯片寄存器级调试<br>`NPU` 内存直接映射与 AI 推理加速 |

---

## 📝 深度文章 / Tech Posts

在这里，我记录了 Camera 系统的源码剖析、踩坑经验与架构思考：

<br>

{% for post in site.posts %}
### 📄 [{{ post.title }}]({{ post.url | relative_url }})
<p style="color: #6a737d; font-size: 0.9em; margin-top: -10px;">
  <i>📅 Published on {{ post.date | date: "%B %d, %Y" }}</i>
</p>

> {% if post.excerpt %}
> {{ post.excerpt | strip_html | strip_newlines | truncate: 150 }}
> {% else %}
> 探索底层 Camera 技术细节、源码分析与实战排错经验... 点击阅读详情。
> {% endif %}

<br>
{% endfor %}

---
<div align="center">
  <p style="color: gray; font-size: 0.8em;">Powered by Jekyll & GitHub Pages · Driven by Code</p>
</div>
