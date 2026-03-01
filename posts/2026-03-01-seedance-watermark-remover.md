# 从零搭建视频去水印服务：FFmpeg + Node.js 实战

## 项目背景

最近需要处理一些带水印的视频,市面上的在线工具要么收费,要么效果不理想。于是决定自己搭建一个视频去水印服务,既能满足需求,又能学习 FFmpeg 的实战应用。

**项目地址**: https://seedance-watermark-remover.ai-genesis.app

## 技术选型

### 为什么不用浏览器端 FFmpeg.wasm？

最初考虑过纯前端方案（FFmpeg.wasm）,但有几个问题：
- wasm 文件 30MB+,加载慢
- 浏览器内存限制,处理大视频容易崩溃
- 性能不如原生 FFmpeg

**最终方案**: Node.js 后端 + FFmpeg 原生处理

## 核心功能实现

### 1. 视频上传与预览

```javascript
// 前端：上传视频并加载到 canvas
const video = document.createElement('video');
video.src = URL.createObjectURL(file);
video.onloadedmetadata = () => {
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  ctx.drawImage(video, 0, 0);
};
```

### 2. 水印区域框选

用户在视频预览上拖拽框选水印位置,记录坐标：

```javascript
let startX, startY, endX, endY;
canvas.addEventListener('mousedown', (e) => {
  startX = e.offsetX;
  startY = e.offsetY;
});
canvas.addEventListener('mouseup', (e) => {
  endX = e.offsetX;
  endY = e.offsetY;
  // 计算水印区域
  const x = Math.min(startX, endX);
  const y = Math.min(startY, endY);
  const w = Math.abs(endX - startX);
  const h = Math.abs(endY - startY);
});
```

### 3. 后端 FFmpeg 处理

```javascript
const ffmpeg = require('fluent-ffmpeg');

app.post('/api/remove-watermark', upload.single('video'), (req, res) => {
  const { x, y, w, h } = req.body;
  const inputPath = req.file.path;
  const outputPath = `outputs/${Date.now()}.mp4`;

  ffmpeg(inputPath)
    .videoFilters(`delogo=x=${x}:y=${y}:w=${w}:h=${h}`)
    .output(outputPath)
    .on('end', () => res.download(outputPath))
    .on('error', (err) => res.status(500).json({ error: err.message }))
    .run();
});
```

## 关键技术点

### FFmpeg delogo 滤镜

`delogo` 是 FFmpeg 内置的去水印滤镜,原理是：
- 分析水印区域周围像素
- 用插值算法填充水印区域
- 适合简单文字/半透明 Logo

**局限性**: 不是真正的 AI 修复,对复杂水印效果有限。

### 磁盘空间管理

视频文件占用空间大,必须定期清理：

```javascript
const cleanupOldFiles = (dir, maxSize, maxAge) => {
  const files = fs.readdirSync(dir).map(f => ({
    name: f,
    path: path.join(dir, f),
    time: fs.statSync(path.join(dir, f)).mtime.getTime()
  }));

  // 删除超过时间限制的文件
  const now = Date.now();
  files.forEach(f => {
    if (now - f.time > maxAge) {
      fs.unlinkSync(f.path);
    }
  });

  // 如果总大小超限,删除最老的文件
  let totalSize = files.reduce((sum, f) => sum + fs.statSync(f.path).size, 0);
  files.sort((a, b) => a.time - b.time);
  while (totalSize > maxSize && files.length > 0) {
    const oldest = files.shift();
    totalSize -= fs.statSync(oldest.path).size;
    fs.unlinkSync(oldest.path);
  }
};

// 每10分钟清理一次
setInterval(() => {
  cleanupOldFiles('uploads', 5 * 1024 * 1024 * 1024, 30 * 60 * 1000); // 5GB, 30分钟
  cleanupOldFiles('outputs', 10 * 1024 * 1024 * 1024, 2 * 60 * 60 * 1000); // 10GB, 2小时
}, 10 * 60 * 1000);
```

### HTTPS 部署

使用 Let's Encrypt 免费证书：

```bash
# 安装 Certbot
sudo apt install certbot

# 申请证书
sudo certbot certonly --standalone -d seedance-watermark-remover.ai-genesis.app

# Node.js 加载证书
const https = require('https');
const fs = require('fs');

const options = {
  key: fs.readFileSync('/etc/letsencrypt/live/domain/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/domain/fullchain.pem')
};

https.createServer(options, app).listen(443);
```

### PM2 进程管理

```bash
# 启动服务
pm2 start server.js --name seedance-watermark-remover

# 开机自启
pm2 startup
pm2 save

# 监控
pm2 monit
```

## 实际效果

✅ **优点**:
- 处理速度快（后端 FFmpeg 性能强）
- 无需等待 wasm 加载
- 支持大文件处理
- 自动清理磁盘空间

⚠️ **局限**:
- 对简单水印效果好（文字、半透明 Logo）
- 复杂水印需要更高级 AI 模型
- 不支持批量处理（可优化）

## 后续优化方向

1. **集成 AI 修复模型**: 使用 OpenCV inpainting 或深度学习模型
2. **批量处理**: 支持一次上传多个视频
3. **实时进度**: WebSocket 推送处理进度
4. **参数调优**: 让用户自定义 delogo 参数

## 总结

这个项目让我深入理解了：
- FFmpeg 视频处理的实战应用
- Node.js 文件上传与流处理
- 磁盘空间管理策略
- HTTPS 证书配置

虽然 `delogo` 滤镜不是完美方案,但对于简单场景已经够用。如果需要更高质量的去水印效果,可以考虑集成 AI 模型,这也是下一步的优化方向。

---

**项目访问**: https://seedance-watermark-remover.ai-genesis.app  
**技术栈**: Node.js + Express + FFmpeg + PM2  
**部署**: Ubuntu + Let's Encrypt + PM2

