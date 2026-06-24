---
layout: default
title: "Camera Learning Lab - Yang Bolin"
---

<div markdown="0">
<style>
/* ================= 全局现代科技感样式定义 ================= */
:root {
  --primary-gradient: linear-gradient(135deg, #0f2027, #203a43, #2c5364);
  --lens-blue-gradient: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  --tech-cyan: #00f2fe;
  --tech-blue: #4facfe;
  --bg-card: rgba(255, 255, 255, 0.95);
  --border-color: rgba(79, 172, 254, 0.15);
  --text-main: #2c3e50;
  --text-muted: #7f8c8d;
}
<!---->
/* 页面容器宽度优化 */
.wrapper {
  max-width: 1000px !important;
}
<!---->
/* ================= 1. 赛博质感 Hero Banner ================= */
.hero-banner {
  background: var(--primary-gradient);
  border-radius: 20px;
  padding: 40px 30px;
  color: #ffffff;
  position: relative;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(42, 82, 152, 0.15);
  margin-bottom: 35px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}
<!---->
.hero-banner::after {
  content: "";
  position: absolute;
  top: -50%;
  right: -30%;
  width: 400px;
  height: 400px;
  background: radial-gradient(circle, rgba(0, 242, 254, 0.15) 0%, transparent 70%);
  border-radius: 50%;
  pointer-events: none;
}
<!---->
.hero-banner h1 {
  font-size: 2.4rem !important;
  font-weight: 800 !important;
  margin: 0 0 10px 0 !important;
  letter-spacing: 1px;
  background: linear-gradient(to right, #ffffff, #00f2fe) !important;
  -webkit-background-clip: text !important;
  -webkit-text-fill-color: transparent !important;
  display: inline-block;
  color: #fff !important;
}
<!---->
.hero-subtitle {
  font-size: 1.15rem;
  color: rgba(255, 255, 255, 0.85);
  margin: 10px 0 25px 0;
  line-height: 1.6;
}
<!---->
/* 核心能力标签 */
.tech-tag-group {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
<!---->
.tech-badge {
  background: rgba(255, 255, 255, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.25);
  padding: 4px 12px;
  border-radius: 50px;
  font-size: 0.82rem;
  color: #fff;
  font-family: 'Courier New', Courier, monospace;
  transition: all 0.3s ease;
  backdrop-filter: blur(5px);
}
<!---->
.tech-badge:hover {
  background: rgba(0, 242, 254, 0.2);
  border-color: var(--tech-cyan);
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0, 242, 254, 0.2);
}
<!---->
/* ================= 2. 车载 Camera 实战特写 (DVR & 360 AVM) ================= */
.car-lab-section {
  margin-bottom: 40px;
}
<!---->
.car-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 20px;
  margin-top: 15px;
}
<!---->
.car-card {
  background: var(--bg-card);
  border: 1px solid var(--border-color);
  border-radius: 16px;
  padding: 24px;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.03);
}
<!---->
.car-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 24px rgba(42, 82, 152, 0.1);
  border-color: var(--tech-blue);
}
<!---->
.card-icon {
  font-size: 2rem;
  margin-bottom: 12px;
  display: inline-block;
}
<!---->
.car-card h3 {
  margin: 0 0 10px 0 !important;
  font-size: 1.25rem !important;
  color: #1a252f !important;
  font-weight: 700;
}
<!---->
.car-card p {
  font-size: 0.9rem !important;
  color: var(--text-muted);
  line-height: 1.5;
  margin: 0;
}
<!---->
/* ================= 3. 技术栈多维矩阵 ================= */
.matrix-section {
  margin-bottom: 40px;
}
<!---->
.matrix-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 15px;
}
<!---->
@media (max-width: 768px) {
  .matrix-grid {
    grid-template-columns: 1fr;
  }
}
<!---->
.matrix-box {
  background: #fdfdfd;
  border-left: 4px solid var(--tech-blue);
  border-radius: 4px 12px 12px 4px;
  padding: 15px 20px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.02);
}
<!---->
.matrix-box h4 {
  margin: 0 0 8px 0 !important;
  font-size: 1rem !important;
  color: #2c3e50;
  font-weight: 700;
}
<!---->
.matrix-box ul {
  padding-left: 15px;
  margin: 0;
  font-size: 0.85rem;
  color: var(--text-muted);
}
<!---->
.matrix-box li {
  margin-bottom: 5px;
}
<!---->
/* ================= 4. 精美文章卡片列表 ================= */
.posts-section {
  margin-top: 40px;
}
<!---->
.section-title {
  font-size: 1.5rem !important;
  font-weight: 700 !important;
  color: #1a252f !important;
  margin-bottom: 20px !important;
  display: flex;
  align-items: center;
  gap: 10px;
}
<!---->
.post-card-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}
<!---->
.post-item-card {
  background: #ffffff;
  border: 1px solid rgba(0,0,0,0.05);
  border-radius: 12px;
  padding: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: all 0.2s ease;
  box-shadow: 0 2px 6px rgba(0,0,0,0.01);
  text-decoration: none !important;
}
<!---->
.post-item-card:hover {
  background: #fafcff;
  border-color: rgba(79, 172, 254, 0.3);
  transform: scale(1.01);
}
<!---->
.post-meta-left {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
<!---->
.post-card-title {
  font-size: 1.15rem !important;
  font-weight: 700 !important;
  color: #2a5298 !important;
  margin: 0 !important;
  transition: color 0.2s ease;
}
<!---->
.post-item-card:hover .post-card-title {
  color: #00f2fe !important;
  background: linear-gradient(to right, #2a5298, #00f2fe);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
<!---->
.post-card-desc {
  font-size: 0.88rem;
  color: var(--text-muted);
  margin: 0;
}
<!---->
.post-meta-right {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 6px;
}
<!---->
.post-card-date {
  font-size: 0.82rem;
  font-family: 'Courier New', Courier, monospace;
  color: var(--text-muted);
  background: #f1f2f6;
  padding: 2px 8px;
  border-radius: 4px;
}
<!---->
.post-card-readmore {
  font-size: 0.85rem;
  color: var(--tech-blue);
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 4px;
}
</style>
<!---->
<!-- ================= 1. HERO BANNER 区 ================= -->
<div class="hero-banner">
  <div class="hero-title-area">
    <h1>Camera Learning Lab</h1>
  
</div>


  <p class="hero-subtitle">
    👋 你好，我是 <b>Yang Bolin</b>。专注于车载智能视界开发。深耕 <b>Android Camera FW 框架层、HAL3 标准规范以及底层核心驱动接口</b>。在这里，记录我打通硬件到 AI 神经网络的高速链路。
  </p>
  <div class="tech-tag-group">
    <span class="tech-badge">⚡ Android 12/13/14</span>
    <span class="tech-badge">🛠️ Camera HAL3 (AIDL)</span>
    <span class="tech-badge">🧠 RK3562 / NPU (RKNN)</span>
    <span class="tech-badge">🏎️ TP2856 / TP9951</span>
    <span class="tech-badge">🚀 dma-buf / Zero-Copy</span>
    <span class="tech-badge">📦 C++/JNI/NDK</span>
  </div>
</div>

<!-- ================= 2. 车载 CAMERA 实战特写区 ================= -->
<div class="car-lab-section">
  <h2 class="section-title">🚗 车载 Camera 核心实战战地</h2>
  <div class="car-grid">
    <div class="car-card">
      <span class="card-icon">📹</span>
      <h3>智能 DVR (行车记录仪)</h3>
      <p>基于 <b>TP2856</b> 的 4 路 AHD 模拟高清汇聚采集方案。打通瑞芯微 <b>RK-CIF</b> 驱动，配置 MIPI 虚拟通道接收，实现超高稳定性的多路 YUV 并行硬件 MPP 编码录像。</p>
    </div>
    <div class="car-card">
      <span class="card-icon">🔄</span>
      <h3>360 AVM (全景环视)</h3>
      <p>控制 4 路广角摄像头帧同步采集。优化 HAL3 内存机制，利用 <b>OpenGL ES OES 外部纹理</b> 与 EGLImage 扩展实现 0 次 CPU 拷贝，为 3D 环视算法输送极致性能的数据流。</p>
    </div>
    <div class="car-card">
      <span class="card-icon">🧠</span>
      <h3>AI 视觉对接 (ADAS/BSD)</h3>
      <p>打通图像采集与 <b>NPU (Rockchip RKNN)</b> 的物理共享通道。利用 <b>dma-buf fd</b>实现一帧不落、15ms 延迟的主动安全智能识别管道。</p>
    </div>
  </div>
</div>

<!-- ================= 3. 技术栈多维矩阵区 ================= -->
<div class="matrix-section">
  <h2 class="section-title">📊 核心研究体系</h2>
  <div class="matrix-grid">
    <div class="matrix-box" style="border-left-color: #4facfe;">
      <h4>应用与 FW 层</h4>
      <ul>
        <li>Camera2/CameraX 控制栈</li>
        <li>cameraserver 核心管理</li>
        <li>JNI 数据强转与脱壳</li>
        <li>多应用权限抢占策略</li>
      </ul>
    </div>
    <div class="matrix-box" style="border-left-color: #00f2fe;">
      <h4>HAL & 驱动层</h4>
      <ul>
        <li>Camera HAL3 AIDL/HIDL</li>
        <li>BufferQueue 状态机流转</li>
        <li>V4L2 驱动机制与 ioctl</li>
        <li>MIPI Virtual Channel</li>
      </ul>
    </div>
    <div class="matrix-box" style="border-left-color: #2a5298;">
      <h4>底层控制与硬件</h4>
      <ul>
        <li>ION/Gralloc 内存共享</li>
        <li>TP2856/TP9951 解码寄存器</li>
        <li>dma-buf 零拷贝技术</li>
        <li>NPU 内存直接映射</li>
      </ul>
    </div>
  </div>
</div>

<!-- ================= 4. 精美文章卡片列表区 ================= -->
<div class="posts-section">
  <h2 class="section-title">📝 最新技术沉淀</h2>
  <div class="post-card-list">
    {% for post in site.posts %}
    <a href="{{ post.url | relative_url }}" class="post-item-card">
      <div class="post-meta-left">
        <h3 class="post-card-title">{{ post.title }}</h3>
        <p class="post-card-desc">
          {% if post.excerpt %}
            {{ post.excerpt | strip_html | strip_newlines | truncate: 80 }}
          {% else %}
            点击深入探索此篇底层 Camera 技术细节...
          {% endif %}
        </p>
      </div>
      <div class="post-meta-right">
        <span class="post-card-date">{{ post.date | date: "%Y-%m-%d" }}</span>
        <span class="post-card-readmore">阅读全文 →</span>
      </div>
    </a>
    {% endfor %}
  </div>
</div>
</div>
