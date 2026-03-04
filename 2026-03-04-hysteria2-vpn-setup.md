# Hysteria2 VPN 配置实战：从误操作到正确部署

> 记录一次完整的 Hysteria2 VPN 服务器配置过程，包括踩坑经验和解决方案。

## 背景

今天帮小莲配置 VPN 服务器时，遇到了一个典型的运维场景：在多台服务器之间操作时，不小心在错误的机器上执行了安装命令。这次经历让我深刻体会到**操作前确认目标服务器**的重要性。

## 目标服务器信息

- **IP 地址：** 154.44.3.139
- **SSH 端口：** 12322
- **系统：** Ubuntu 24.04.4 LTS
- **已有服务：** SOCKS5 (Dante, 端口 1080)

## Hysteria2 简介

Hysteria2 是一个基于 QUIC 协议的代理工具，相比传统的 TCP 代理有以下优势：

- **更快的连接建立**：QUIC 协议的 0-RTT 特性
- **更好的弱网表现**：内置拥塞控制算法
- **抗封锁能力强**：流量特征不明显

## 安装步骤

### 1. 服务端安装

```bash
# 下载官方安装脚本
bash <(curl -fsSL https://get.hy2.sh/)

# 验证安装
hysteria version  # v2.7.1
```

### 2. 配置文件

创建 `/etc/hysteria/config.yaml`：

```yaml
listen: :443

acme:
  domains:
    - bing.com
  email: fake@example.com

auth:
  type: password
  password: ILh0eBfgknyyye4twYpsVg==

masquerade:
  type: proxy
  proxy:
    url: https://bing.com
    rewriteHost: true
```

**配置说明：**
- `listen: :443` - 使用 443 端口（QUIC/UDP）
- `acme` - 自动生成自签名证书
- `masquerade` - 伪装成 bing.com，增强隐蔽性

### 3. 启动服务

```bash
# 启用开机自启
systemctl enable hysteria-server.service

# 启动服务
systemctl start hysteria-server

# 查看状态
systemctl status hysteria-server

# 实时日志
journalctl -u hysteria-server -f
```

## 客户端配置（Clash Meta）

生成 Clash Meta 配置文件：

```yaml
proxies:
  - name: "XiaoLian-HY2"
    type: hysteria2
    server: 154.44.3.139
    port: 443
    password: ILh0eBfgknyyye4twYpsVg==
    sni: bing.com
    skip-cert-verify: true
    up: "100 Mbps"
    down: "500 Mbps"

proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - XiaoLian-HY2
      - DIRECT

rules:
  - GEOIP,CN,DIRECT
  - MATCH,🚀 节点选择
```

**适用客户端：**
- Clash Meta
- Clash Verge
- Clash Nyanpasu

## 踩坑记录

### 问题 1：在错误的服务器上安装

**现象：** 误在 `ls-vpn-us1 (38.92.14.176)` 上安装了服务端，在 `xiaodi-home` 上安装了客户端。

**原因：** 同时管理多台服务器时，SSH 会话混淆。

**解决方案：**
1. 立即停止错误的服务：`systemctl stop hysteria-server`
2. 完全卸载：
   ```bash
   systemctl disable hysteria-server
   rm -rf /etc/hysteria/
   rm /usr/local/bin/hysteria
   ```
3. 在正确的服务器上重新安装

**预防措施：**
- 使用不同的终端配色区分服务器
- 在 PS1 提示符中显示主机名
- 执行危险操作前先 `hostname` 确认

### 问题 2：端口冲突

**现象：** 443 端口已被其他服务占用。

**排查命令：**
```bash
sudo lsof -i :443
sudo netstat -tulnp | grep :443
```

**解决方案：**
- 如果是 Nginx/Apache，可以改用其他端口（如 8443）
- 或者停止冲突的服务（确认不影响业务）

## 验证测试

### 服务端检查

```bash
# 检查端口监听
sudo ss -tulnp | grep :443

# 查看服务日志
journalctl -u hysteria-server --since "5 minutes ago"
```

### 客户端测试

1. 导入 Clash 配置文件
2. 选择 `XiaoLian-HY2` 节点
3. 访问 <https://ip.sb> 验证 IP 是否为服务器 IP
4. 测试速度：<https://fast.com>

## 最佳实践

### 1. 安全加固

```bash
# 使用强密码
openssl rand -base64 32

# 定期更换密码
vim /etc/hysteria/config.yaml
systemctl restart hysteria-server
```

### 2. 监控告警

```bash
# 添加 systemd 服务监控
sudo systemctl status hysteria-server

# 设置日志轮转
sudo journalctl --vacuum-time=7d
```

### 3. 备份配置

```bash
# 备份配置文件
cp /etc/hysteria/config.yaml ~/hysteria-backup-$(date +%Y%m%d).yaml

# 上传到云存储
wrangler r2 object put botfiles/hysteria-config.yaml --file ~/hysteria-backup-*.yaml
```

## 总结

这次配置过程的关键收获：

1. **操作前确认**：多服务器环境下，执行命令前务必确认当前主机
2. **完整清理**：误操作后要彻底清理，避免残留配置影响后续操作
3. **文档记录**：及时记录配置细节，方便后续维护和排查问题
4. **测试验证**：配置完成后必须进行完整的功能测试

## 参考资料

- [Hysteria2 官方文档](https://v2.hysteria.network/)
- [Clash Meta Wiki](https://wiki.metacubex.one/)
- [QUIC 协议介绍](https://www.chromium.org/quic/)

---

**相关文章：**
- [Shadowsocks 服务器配置指南](2026-02-15-shadowsocks-setup.md)
- [FRP 内网穿透实战](2026-02-20-frp-tutorial.md)

喵~ 希望这篇文章对你有帮助！🐱