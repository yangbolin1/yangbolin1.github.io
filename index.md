---
layout: default
title: "Camera Learning Lab - Yang Bolin"
---

<style>
/* ================= 极简极客风双栏布局 ================= */
.blog-container {
  display: flex;
  gap: 40px;
  margin-top: 30px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}

/* 左侧文章主栏 */
.blog-main {
  flex: 2.3;
  min-width: 0;
}

/* 右侧侧边栏 */
.blog-sidebar {
  flex: 1;
  min-width: 260px;
  padding-left: 25px;
  border-left: 1px solid #e1e4e8;
}

@media (max-width: 768px) {
  .blog-container {
    flex-direction: column;
  }
  .blog-sidebar {
    border-left: none;
    border-top: 1px solid #e1e4e8;
    padding-left: 0;
    padding-top: 25px;
  }
}

/* 文章列表样式 */
.section-title {
  font-size: 1.3rem;
  font-weight: 600;
  color: #24292e;
  margin-top: 0;
  margin-bottom: 20px;
  padding-bottom: 8px;
  border-bottom: 1px solid #e1e4e8;
}

.post-item {
  margin-bottom: 28px;
  padding-bottom: 20px;
  border-bottom: 1px dashed #e1e4e8;
}

.post-item:last-child {
  border-bottom: none;
}

.post-title {
  font-size: 1.2rem !important;
  font-weight: 600 !important;
  margin: 0 0 6px 0 !important;
}

.post-title a {
  color: #0366d6 !important;
  text-decoration: none !important;
}

.post-title a:hover {
  text-decoration: underline !important;
}

.post-meta {
  font-size: 0.82rem;
  color: #586069;
  margin-bottom: 10px;
  font-family: "SFMono-Regular", Consolas, monospace;
}

.post-excerpt {
  font-size: 0.92rem;
  color: #444d56;
  line-height: 1.5;
}

/* 侧边栏个人名片 */
.profile-card h3 {
  margin-top: 0;
  font-size: 1.15rem;
  color: #24292e;
}

.profile-card p {
  font-size: 0.88rem;
  color: #586069;
  line-height: 1.6;
}

.tech-stack {
  margin-top: 30px;
}

.tech-stack h4 {
  font-size: 0.9rem;
  color: #24292e;
  margin-bottom: 12px;
}

.tag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.tag-item {
  font-size: 0.75rem;
  background-color: #f1f8ff;
  color: #0366d6;
  padding: 3px 10px;
  border-radius: 3px;
  font-family: "SFMono-Regular", Consolas, monospace;
  border: 1px solid rgba(3, 102, 214, 0.1);
}
</style>

<div class="blog-container">
<!---->
  <!-- ================= 左侧：技术文章列表 ================= -->
  <div class="blog-main">
    <h2 class="section-title">最新文章 / Tech Posts</h2>
    <div class="post-list">
      {% for post in site.posts %}
      <div class="post-item">
        <h3 class="post-title">
          <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
        </h3>
        <div class="post-meta">
          <span>📅 发布于 {{ post.date | date: "%Y-%m-%d" }}</span>
        
</div>


        <div class="post-excerpt">
          {% if post.excerpt %}
            {{ post.excerpt | strip_html | strip_newlines | truncate: 120 }}
          {% else %}
            点击阅读这篇关于 Camera 底层技术的沉淀与实战记录...
          {% endif %}
        </div>
      </div>
      {% endfor %}
    </div>
  </div>

  <!-- ================= 右侧：侧边栏个人信息 ================= -->
  <div class="blog-sidebar">
    <div class="profile-card">
      <h3>👋 Yang Bolin</h3>
      <p>Android Camera 系统开发工程师，专注于车机 DVR 录像、360 环视拼接（AVM）底层技术。致力于打通图像采集至 NPU 神经网络的极速零拷贝（Zero-Copy）数据管道。</p>
    </div>

    <div class="tech-stack">
      <h4>🛠️ 技术栈 / Tech Stack</h4>
      <div class="tag-list">
        <span class="tag-item">Android FW</span>
        <span class="tag-item">Camera HAL3</span>
        <span class="tag-item">C++ / JNI</span>
        <span class="tag-item">RK3562</span>
        <span class="tag-item">TP2856 / TP9951</span>
        <span class="tag-item">dma-buf</span>
      </div>
    </div>
  </div>

</div>
