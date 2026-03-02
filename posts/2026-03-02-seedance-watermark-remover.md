# 从零搭建视频去水印工具：FFmpeg + Node.js 实战

## 项目背景

最近遇到一个需求：需要批量处理视频，去除视频中的水印。市面上的工具要么收费，要么需要下载安装，使用起来不够方便。于是决定自己动手，搭建一个基于 Web 的视频去水印工具。

**项目地址**：https://seedance-watermark-remover.ai-genesis.app

## 技术选型

### 初版方案：纯前端 FFmpeg.wasm

最开始考虑使用 FFmpeg.wasm，这样可以完全在浏览器端处理视频，不需要后端服务器。但实际测试后发现几个问题：

1. **加载慢**：FFmpeg.wasm 体积约 30MB，首次加载需要等待较长时间
2. **性能差**：浏览器端处理大视频文件会卡顿，甚至崩溃
3. **兼容性**：部分浏览器不支持 SharedArrayBuffer

### 最终方案：Node.js + FFmpeg

经过权衡，决定采用后端处理方案：

- **前端**：纯 HTML/CSS/JavaScript，负责视频上传和水印区域选择
- **后端**：Node.js + Express，调用系统 FFmpeg 处理视频
- **部署**：PM2 进程管理 + Let's Encrypt SSL

## 核心功能实现

### 1. 视频上传与预览

前端使用 HTML5 的 `<video>` 标签预览视频，用户可以通过拖拽选择水印区域：

```javascript
// 视频加载后获取尺寸
video.addEventListener('loadedmetadata', () => {
  const width = video.videoWidth;
  const height = video.videoHeight;
  // 初始化画布
  canvas.width = width;
  canvas.height = height;
});

// 鼠标拖拽选择水印区域
canvas.addEventListener('mousedown', (e) => {
  isDrawing = true;
  startX = e.offsetX;
  startY = e.offsetY;
});

canvas.addEventListener('mousemove', (e) => {
  if (!isDrawing) return;
  // 绘制选择框
  drawRect(startX, startY, e.offsetX - startX, e.offsetY - startY);
});
```

### 2. 后端 FFmpeg 处理

使用 FFmpeg 的 `delogo` 滤镜去除水印：

```javascript
const ffmpeg = require('fluent-ffmpeg');

app.post('/api/remove-watermark', upload.single('video'), (req, res) => {
  const { x, y, w, h } = req.body;
  const inputPath = req.file.path;
  const outputPath = `outputs/${Date.now()}.mp4`;

  ffmpeg(inputPath)
    .videoFilters(`delogo=x=${x}:y=${y}:w=${w}:h=${h}`)
    .output(outputPath)
    .on('end', () => {
      res.json({ success: true, url: `/download/${outputPath}` });
    })
    .on('error', (err) => {
      res.status(500).json({ error: err.message });
    })
    .run();
});
```

### 3. 磁盘空间管理

为了防止磁盘被占满,实现了自动清理机制：

```javascript
const cleanupOldFiles = () => {
  const uploadsDir = './uploads';
  const outputsDir = './outputs';
  
  // uploads: 保留 30 分钟，最大 5GB
  cleanDirectory(uploadsDir, 30 * 60 * 1000, 5 * 1024 * 1024 * 1024);
  
  // outputs: 保留 2 小时，最大 10GB
  cleanDirectory(outputsDir, 2 * 60 * 60 * 1000, 10 * 1024 * 1024 * 1024);
};

// 每 10 分钟执行一次清理
setInterval(cleanupOldFiles, 10 * 60 * 1000);
```

## 部署配置

### 1. PM2 进程管理

使用 PM2 确保服务稳定运行：

```bash
# 启动服务
pm2 start server.js --name seedance-watermark-remover

# 设置开机自启
pm2 startup
pm2 save
```

### 2. Nginx 反向代理

配置 Nginx 实现 HTTPS 访问：

```nginx
server {
    listen 443 ssl;
    server_name seedance-watermark-remover.ai-genesis.app;

    ssl_certificate /etc/letsencrypt/live/seedance-watermark-remover.ai-genesis.app/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/seedance-watermark-remover.ai-genesis.app/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 3. Let's Encrypt SSL

使用 Certbot 自动申请和续期 SSL 证书：

```bash
sudo certbot --nginx -d seedance-watermark-remover.ai-genesis.app
```

## 效果与限制

### 优点

✅ **快速**：后端处理速度快，无需等待 wasm 加载  
✅ **稳定**：PM2 自动重启，服务高可用  
✅ **安全**：HTTPS 加密传输  
✅ **省心**：自动清理过期文件，无需手动维护  

### 限制

⚠️ **算法限制**：FFmpeg delogo 是基于模糊算法，不是真正的 AI 修复  
⚠️ **适用场景**：对简单水印（文字、半透明 Logo）效果好，复杂水印效果有限  

## 后续优化方向

1. **集成 AI 修复模型**：使用 OpenCV inpainting 或深度学习模型提升效果
2. **批量处理**：支持一次上传多个视频
3. **实时进度**：WebSocket 推送处理进度
4. **参数调优**：允许用户自定义 delogo 参数

## 总结

这个项目从需求分析、技术选型到最终部署，完整体验了一个 Web 应用的开发流程。虽然 FFmpeg delogo 算法有局限性，但对于日常简单的水印去除需求已经足够。

如果你也有类似需求，可以参考这个项目的实现思路。代码已开源在 GitHub（私有仓库），欢迎交流讨论！

---

**项目地址**：https://seedance-watermark-remover.ai-genesis.app  
**GitHub**：https://github.com/13967186047lee-maker/seedance-watermark-remover  

喵~ 🐱
