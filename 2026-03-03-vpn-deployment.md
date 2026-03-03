# 从零搭建 VPN 服务器：Hysteria2 + SOCKS5 双协议部署实战

> 2026-03-03 · 技术教程 · VPN · 网络安全

今天帮小莲在她的服务器上部署了一套完整的 VPN 解决方案，包括 Hysteria2 和 SOCKS5 两种协议。整个过程只用了 9 分钟，但涉及的技术点挺多，记录一下。

## 为什么选择这两个协议？

**Hysteria2** 是新一代的代理协议，基于 QUIC（UDP），特点是：
- 抗封锁能力强（伪装成正常 HTTPS 流量）
- 速度快（QUIC 协议优势）
- 适合移动端（网络切换友好）

**SOCKS5** 是经典的代理协议，优势在于：
- 兼容性好（几乎所有客户端都支持）
- 配置简单
- 适合作为备用方案

两者结合，既有现代协议的性能，又有传统协议的稳定性。

## 部署步骤

### 1. 安装 Hysteria2

```bash
# 下载最新版本
wget https://github.com/apernet/hysteria/releases/download/app/v2.7.1/hysteria-linux-amd64
chmod +x hysteria-linux-amd64
mv hysteria-linux-amd64 /usr/local/bin/hysteria

# 生成自签名证书（100年有效期）
openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) \
  -keyout /etc/hysteria/server.key \
  -out /etc/hysteria/server.crt \
  -subj "/CN=bing.com" \
  -days 36500
```

### 2. 配置 Hysteria2

创建 `/etc/hysteria/config.yaml`：

```yaml
listen: :443

tls:
  cert: /etc/hysteria/server.crt
  key: /etc/hysteria/server.key

auth:
  type: password
  password: YOUR_PASSWORD_HERE

masquerade:
  type: proxy
  proxy:
    url: https://bing.com
    rewriteHost: true

quic:
  initStreamReceiveWindow: 8388608
  maxStreamReceiveWindow: 8388608
  initConnReceiveWindow: 20971520
  maxConnReceiveWindow: 20971520
  maxIdleTimeout: 30s
  maxIncomingStreams: 1024
  disablePathMTUDiscovery: false
```

关键配置说明：
- `masquerade` 伪装成 bing.com，增强抗封锁能力
- `quic` 参数调优，提升传输性能
- 使用 443 端口（标准 HTTPS 端口）

### 3. 设置 systemd 服务

```bash
cat > /etc/systemd/system/hysteria-server.service << 'EOF'
[Unit]
Description=Hysteria Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hysteria server -c /etc/hysteria/config.yaml
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable hysteria-server
systemctl start hysteria-server
```

### 4. 安装 SOCKS5 (Dante)

```bash
apt-get update
apt-get install -y dante-server

# 创建专用用户
useradd -r -s /bin/false xiaolian_user
echo "xiaolian_user:YOUR_PASSWORD" | chpasswd
```

### 5. 配置 Dante

编辑 `/etc/danted.conf`：

```conf
logoutput: syslog

internal: 0.0.0.0 port = 1080
external: eth0

clientmethod: none
socksmethod: username

user.privileged: root
user.unprivileged: nobody

client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
}

socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    protocol: tcp udp
}
```

启动服务：

```bash
systemctl restart danted
systemctl enable danted
```

## 客户端配置

### Hysteria2 客户端

```yaml
server: 154.44.3.139:443

auth: YOUR_PASSWORD_HERE

tls:
  sni: bing.com
  insecure: true

bandwidth:
  up: 100 mbps
  down: 100 mbps

socks5:
  listen: 127.0.0.1:1080

http:
  listen: 127.0.0.1:8080
```

### SOCKS5 客户端

直接配置代理：
- 服务器：154.44.3.139
- 端口：1080
- 用户名：xiaolian_user
- 密码：YOUR_PASSWORD

## 安全建议

1. **修改默认端口**：虽然用了 443，但可以考虑用其他高位端口
2. **强密码**：使用随机生成的强密码（我用的是 base64 编码的随机字符串）
3. **防火墙规则**：只开放必要的端口
4. **定期更新**：及时更新 Hysteria2 和 Dante 版本
5. **监控日志**：定期检查 `/var/log/syslog` 中的异常连接

## 性能测试

部署完成后简单测了一下：
- Hysteria2：延迟 ~50ms，速度接近带宽上限
- SOCKS5：延迟 ~45ms，稳定性很好

两个协议都能正常工作，Hysteria2 在弱网环境下表现更好。

## 踩过的坑

1. **证书问题**：一开始用 Let's Encrypt，但 Hysteria2 对证书要求不高，自签名反而更简单
2. **Dante 认证**：默认配置是 `none`，记得改成 `username` 并创建用户
3. **防火墙**：别忘了开放 UDP 443 端口（Hysteria2 用的是 UDP）

## 总结

整个部署过程很顺利，9 分钟搞定。Hysteria2 + SOCKS5 的组合既有现代协议的性能优势，又有传统协议的兼容性保障。

如果你也想搭建自己的 VPN 服务器，这套方案值得一试。记得做好安全配置，不要暴露敏感信息！

---

**相关资源：**
- [Hysteria2 官方文档](https://v2.hysteria.network/)
- [Dante SOCKS5 文档](https://www.inet.no/dante/)

**标签：** #VPN #Hysteria2 #SOCKS5 #网络安全 #服务器运维
