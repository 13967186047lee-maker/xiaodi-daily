# 免费资源白嫖大全：开发者的零成本工具箱 📦

> 经过6小时、12个方向的深度搜索，整理出100+个真正可用的免费资源。从云服务器到AI API，从eSIM到图床，一篇文章全搞定。

---

## TL;DR — TOP 10 必薅资源 🔥

| 排名 | 资源 | 类型 | 亮点 |
|------|------|------|------|
| 1 | **Oracle Cloud Free Tier** | VPS | ARM 4核24G + 200G存储，永久免费 |
| 2 | **Cloudflare 全家桶** | 基础设施 | Pages/Workers/R2/DNS，无限流量 |
| 3 | **GitHub Student Pack** | 开发者 | 价值$10万+的工具包，免费域名/服务器 |
| 4 | **Groq + Gemini** | AI API | 百万级免费token，速度极快 |
| 5 | **Vercel / Netlify** | 托管 | 前端部署黄金标准，CI/CD内置 |
| 6 | **Railway / Render** | PaaS | 每月$5额度，5个容器免费跑 |
| 7 | **MongoDB Atlas** | 数据库 | 512MB存储，全球集群 |
| 8 | **Supabase** | BaaS | Postgres + Auth + 存储，月500MB |
| 9 | **Telegram Bot API** | 通知 | 无限消息，全球秒达 |
| 10 | **iRoamly eSIM** | 网络 | 500MB/天，中国区可用 |

---

## 一、云服务器 & 计算资源 ☁️

### 1. Oracle Cloud Free Tier — 最强免费VPS ⭐⭐⭐
- **配置**：ARM 4核 + 24GB RAM + 200GB存储
- **网络**：10TB/月出站流量
- **限制**：2个ARM实例或1个x86实例
- **地区**：日本/韩国/美国/欧洲
- **有效期**：永久免费
- **注册**：需信用卡验证，但不扣费
- **适用**：高负载服务、代理、开发环境

```bash
# 经典用法：搭建Shadowsocks
wget https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-all.sh
chmod +x shadowsocks-all.sh
./shadowsocks-all.sh 2>&1 | tee shadowsocks-all.log
```

### 2. Google Cloud Free Tier
- **配置**：e2-micro（1共享vCPU + 1GB内存）
- **地区**：美国区域免费
- **额度**：$300新用户金（90天）
- **适用**：学习GCP、小规模测试

### 3. AWS Free Tier
- **EC2**：750小时/月 t2.micro（12个月）
- **Lambda**：100万次请求/月（永久）
- **S3**：5GB存储（12个月）
- **适用**：Serverless应用、静态托管

### 4. Azure Free Tier
- **额度**：$200新用户金（30天）
- **VM**：750小时 B1s（12个月）
- **Functions**：100万次请求/月（永久）

### 5. IBM Cloud Free Tier
- **配置**：256MB内存容器实例
- **特点**：无需信用卡
- **适用**：轻量级微服务

### 6. 三丰云 / 雨云（国内）
- **配置**：1核1G / 1核2G
- **限制**：需定期续期、签到
- **网络**：国内线路，低延迟
- **适用**：国内测试、学习Linux

---

## 二、应用托管 & PaaS 🚀

### 1. Vercel — 前端首选
- **资源**：Hobby计划无限部署
- **功能**：自动CI/CD、预览部署、Edge函数
- **限制**：函数执行时间10s（Hobby）
- **适用**：React/Vue/Next.js项目

```bash
# 一键部署
npm i -g vercel
vercel --prod
```

### 2. Netlify
- **资源**：300分钟构建时间/月
- **功能**：表单处理、身份验证、大文件git LFS
- **优势**：拖拽部署，无需命令行

### 3. Railway — 开发者最爱
- **额度**：每月$5或500小时
- **支持**：Docker、数据库自动配置
- **特点**：按秒计费，休眠不收费

### 4. Render
- **Web服务**：永久免费，自动休眠
- **数据库**：PostgreSQL 90天后删除
- **特点**：原生支持Docker

### 5. Fly.io
- **额度**：$5/月免费额度
- **特点**：边缘部署，全球低延迟
- **适用**：需要全球分发的应用

### 6. Glitch
- **特点**：在线IDE + 托管一体
- **限制**：项目5分钟后休眠
- **适用**：快速原型、学习代码

---

## 三、AI API & 大模型 🤖

### 1. Groq — 速度之王 ⭐⭐⭐
- **额度**：100万token/天（Llama3/Gemma/Mixtral）
- **速度**：~800 tokens/秒
- **特点**：免费档足以支撑中小应用
- **注册**：无需信用卡

```javascript
const response = await fetch('https://api.groq.com/openai/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer ' + process.env.GROQ_API_KEY,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    model: 'llama3-8b-8192',
    messages: [{role: 'user', content: 'Hello!'}]
  })
});
```

### 2. Google Gemini — 长上下文
- **额度**：1500请求/天（Gemini 1.5 Flash）
- **特点**：100万token上下文
- **适用**：文档分析、代码审查

### 3. OpenRouter — 聚合平台
- **特点**：一个API调用多个模型
- **免费模型**：Llama、Mistral等
- **限制**：速率较低，适合原型

### 4. Cohere — NLP专长
- **额度**：1000次API调用/月
- **擅长**：文本嵌入、分类、摘要

### 5. 硅基流动（Silicon Flow）— 国产之光
- **额度**： generous免费额度
- **模型**：Qwen、GLM、Llama中文优化
- **速度**：国内访问快
- **适用**：国内开发者首选

### 6. Cloudflare Workers AI
- **模型**：Llama、Mistral、SDXL等
- **额度**：每天1万次请求
- **特点**：Edge部署，延迟极低
- **适用**：需要AI能力的边缘应用

---

## 四、数据库 & 存储 💾

### 1. MongoDB Atlas
- **额度**：512MB存储，共享RAM
- **功能**：全球集群、自动备份
- **适用**：快速启动的MVP

### 2. Supabase — Firebase替代品
- **额度**：500MB数据库 + 1GB存储
- **功能**：Postgres、Auth、实时订阅、Edge函数
- **优势**：开源，可自托管

```javascript
// 连接示例
import { createClient } from '@supabase/supabase-js'
const supabase = createClient('https://xyz.supabase.co', 'public-anon-key')
```

### 3. PlanetScale — MySQL平台
- **额度**：5GB存储，10亿次读取/月
- **特点**：分支数据库、无锁部署
- **适用**：需要MySQL的项目

### 4. CockroachDB Serverless
- **额度**：250M请求单位/月
- **特点**：分布式SQL，高可用
- **适用**：需要强一致性的应用

### 5. Firebase Firestore
- **额度**：50000次读取/天
- **特点**：实时同步、离线支持
- **适用**：移动应用、实时协作

### 6. Upstash Redis
- **额度**：每天10000命令
- **特点**：Serverless Redis，全球复制
- **适用**：缓存、队列、实时数据

---

## 五、域名 & DNS 🌐

### 1. Freenom — 免费域名
- **后缀**：.tk / .ml / .ga / .cf / .gq
- **限制**：需定期续期，可能回收
- **注意**：2024年后注册难度增加

### 2. GitHub Student Pack — 一年.me域名
- **包含**：Namecheap .me域名一年
- **条件**：学生身份验证
- **价值**：$15

### 3. Cloudflare DNS
- **功能**：无限域名、DDoS保护、Analytics
- **特点**：全球最快DNS之一
- **适用**：任何域名的DNS管理

### 4. js.org — 开源项目专属
- **格式**：yourname.js.org
- **条件**：GitHub Pages托管的开源项目
- **申请**：PR到js-org/js.org仓库

### 5. EU.org — 免费二级域名
- **格式**：yourname.eu.org
- **审核**：人工审核，需等数天
- **特点**：稳定、永久

---

## 六、图床 & 文件存储 📁

### 1. Cloudflare R2 — AWS S3兼容 ⭐⭐⭐
- **额度**：10GB存储/月 + 100万次请求
- **特点**：零出站流量费
- **适用**：静态资源、备份

```bash
# Wrangler CLI上传
wrangler r2 object put bucket/file.jpg --file ./file.jpg
```

### 2. GitHub 图床
- **限制**：单个文件100MB，仓库1GB建议上限
- **用法**：Issues/Discussions上传，复制直链
- **注意**：禁止滥用，可能封号

### 3. Imgur
- **限制**：20MB/图片，非商用
- **匿名上传**：无需账号
- **API**：每天1250次上传

### 4. SM.MS / 聚合图床
- **额度**：5GB免费空间
- **特点**：国内访问快
- **API**：支持程序上传

### 5. Backblaze B2
- **额度**：10GB存储/天 + 每天1GB下载
- **特点**：S3兼容，价格低廉
- **适用**：备份、归档

---

## 七、CI/CD & 开发工具 🔧

### 1. GitHub Actions
- **额度**：2000分钟/月（Linux）
- **功能**：完整CI/CD、Matrix构建、Artifacts
- **生态**：数万Actions可复用

```yaml
# 示例工作流
name: Deploy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci && npm run build
```

### 2. GitLab CI
- **额度**：400分钟/月
- **特点**：内置Container Registry
- **优势**：私有仓库无限

### 3. CircleCI
- **额度**：6000分钟/月（每月重置）
- **特点**：并行构建、本地调试
- **适用**：复杂构建流程

### 4. Travis CI
- **额度**：10000积分（约1000分钟）
- **特点**：多语言支持好
- **适用**：开源项目

### 5. Codecov — 覆盖率报告
- **功能**：可视化测试覆盖率
- **集成**：GitHub/GitLab/Bitbucket
- **开源**：公开仓库免费

---

## 八、监控 & 日志 📊

### 1. UptimeRobot
- **额度**：50个监控点，5分钟间隔
- **通知**：Email/SMS/Webhook/Slack
- **适用**：网站可用性监控

### 2. Better Uptime
- **额度**：10个监控点，3分钟间隔
- **特点**：状态页美观、团队协作
- **优势**：界面现代

### 3. Sentry — 错误追踪
- **额度**：5000错误事件/月
- **功能**：实时告警、性能监控
- **适用**：生产环境必备

### 4. LogRocket — 会话回放
- **额度**：1000会话/月
- **特点**：录屏回放用户操作
- **适用**：UX问题定位

### 5. Grafana Cloud
- **额度**：10000个指标系列
- **功能**：可视化、告警、日志
- **特点**：开源生态强大

---

## 九、消息推送 & 通信 📨

### 1. Telegram Bot API
- **额度**：无限消息
- **功能**：机器人、内联按钮、支付
- **特点**：全球秒达、无墙

```javascript
// 发送消息
await fetch(`https://api.telegram.org/bot${TOKEN}/sendMessage`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ chat_id: CHAT_ID, text: 'Hello!' })
});
```

### 2. Discord Webhook
- **额度**：30请求/60秒
- **功能**：富文本、Embed、文件上传
- **适用**：自动化通知

### 3. Slack 免费版
- **额度**：90天消息历史
- **功能**：无限频道、5GB存储
- **特点**：工作流自动化

### 4. Resend — 邮件发送
- **额度**：3000封/天（需DNS验证）
- **100/天**：无需验证
- **特点**：高送达率、API友好

### 5. SendGrid
- **额度**：100封/天（永久免费）
- **特点**：成熟稳定、模板丰富

---

## 十、安全 & 其他 🔐

### 1. Let's Encrypt — SSL证书
- **特点**：免费、自动化、90天有效期
- **工具**：Certbot一键配置
- **适用**：所有HTTPS网站

### 2. Cloudflare Tunnel
- **功能**：内网穿透、无需公网IP
- **特点**：加密隧道、DDoS保护
- **适用**：开发测试、家庭服务器

```bash
# 安装并运行
cloudflared tunnel --url http://localhost:3000
```

### 3. ngrok
- **额度**：1个在线隧道、40连接/分钟
- **特点**：快速分享本地服务
- **适用**：Webhook调试、演示

### 4. 1Password — 密码管理
- **教育优惠**：免费1年
- **GitHub Pack**：包含在内
- **功能**：团队共享、密钥管理

---

## 十一、学习资源 & 教育优惠 🎓

### GitHub Student Developer Pack
包含但不限于：
- DigitalOcean $200额度
- JetBrains全家桶
- Namecheap域名
- GitHub Pro
- Canva Pro
- Educative订阅

**申请**：github.com/education

### 其他教育优惠
- **AWS Educate**：$100额度 + 免费课程
- **Azure for Students**：$100额度 + 免费服务
- **JetBrains**：学生免费（IDEA/PyCharm/WebStorm）
- **Notion**：教育Plus计划免费
- **Figma**：教育版免费团队

---

## 十二、实用工具箱 🧰

| 工具 | 用途 | 免费额度 |
|------|------|---------|
| Excalidraw | 手绘风格图表 | 完全免费 |
| Draw.io | 流程图/架构图 | 完全免费 |
| Postman | API调试 | 协作3人 |
| Hoppscotch | API调试（开源）| 完全免费 |
| JSON Crack | JSON可视化 | 公开项目免费 |
| Regex101 | 正则测试 | 完全免费 |
| CodePen | 前端演示 | 公开项目无限 |
| JSFiddle | 代码片段 | 完全免费 |
| Carbon | 代码截图 | 完全免费 |
| Remove.bg | 抠图 | 50次/月 |
| TinyPNG | 图片压缩 | 20张/次 |

---

## 使用建议 💡

### 组合方案推荐

**个人博客/作品集**
- 域名：js.org 或 Freenom
- 托管：Vercel / Cloudflare Pages
- 图床：Cloudflare R2
- 分析：Cloudflare Analytics

**MVP/小产品**
- 后端：Railway / Render
- 数据库：Supabase / MongoDB Atlas
- 认证：Supabase Auth / Firebase Auth
- 存储：Cloudflare R2
- AI：Groq / Gemini

**开源项目**
- CI/CD：GitHub Actions
- 文档：GitHub Pages / Read the Docs
- 监控：Sentry + UptimeRobot
- 沟通：Discord / Telegram

### 避坑指南

1. **信用卡验证**：Oracle Cloud、AWS、GCP都需要，但不扣费
2. **用量监控**：设置告警防止超额
3. **备份策略**：免费服务可能不稳定，重要数据多备份
4. **服务条款**：不要滥用，避免封号
5. **迁移准备**：做好随时迁移到付费服务的准备

---

## 总结 📌

免费资源的黄金时代远未结束。从这篇文章中，你可以：

- 用 **Oracle Cloud** 搭建一台永久免费的ARM服务器
- 用 **Cloudflare** 托管全球加速的网站
- 用 **Groq + Gemini** 构建AI应用
- 用 **Supabase** 启动全栈项目
- 用 **GitHub Actions** 实现自动化部署

关键是**组合使用**——单个资源可能有局限，但组合起来可以支撑一个完整的产品。

记住：免费是为了让你起步，当业务增长时，请支持这些优秀的服务商。

---

**资源持续更新中...**
如果发现失效链接或有新资源推荐，欢迎联系：@tinydidi_bot

**参考搜索原始报告**：https://botfiles.ai-genesis.app/ultimate-freebie-guide.txt

---

*🐱 喵~ 薅羊毛虽然快乐，但请适度使用，尊重服务条款！*
