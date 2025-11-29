## 🧹 第一步：安装证书
## 步骤一：安装 `acme.sh`

```
# 安装 acme.sh（推荐使用官方脚本）
curl https://get.acme.sh | sh -s email=your@email.com
```

## 步骤二：配置阿里云 DNS API（用于自动验证）
1. 登录 阿里云 RAM 控制台
2. 创建一个 **子用户**（不要用主账号！）
3. 给该用户授权：
    - `AliyunDNSFullAccess`（或最小权限：`dns:DescribeDomainRecords`, `dns:AddDomainRecord`, `dns:DeleteDomainRecord`）
4. 获取 **AccessKey ID** 和 **AccessKey Secret**
## 步骤三：设置环境变量（临时）

```
export Ali_Key="你的_AccessKey_ID"
export Ali_Secret="你的_AccessKey_Secret"
```
## 步骤四：自动申请泛域名证书

```
# 申请 *.aitaking.com 泛域名证书（使用 DNS 验证）
~/.acme.sh/acme.sh --issue --dns dns_ali -d aitaking.com -d '*.aitaking.com'
```
成功后你会看到：
```
Your cert is in: ~/.acme.sh/aitaking.com/aitaking.com.cer
Your key is in:  ~/.acme.sh/aitaking.com/aitaking.com.key
```

##  步骤五：自动安装证书到 DERP 目录
```
mkdir -p $HOME/derp-server/certs

~/.acme.sh/acme.sh --install-cert -d aitaking.com \
  --key-file       $HOME/derp-server/certs/privkey.pem \
  --fullchain-file $HOME/derp-server/certs/fullchain.pem
```

✅ 此命令会：

- 复制证书和私钥
- **后续自动续期时也会自动更新这两个文件**

## 验证是否成功
```
ls -l /root/derp-server/certs/
```
## 步骤六：设置自动续期（已内置）
`acme.sh` 默认每天检查一次，到期前 30 天自动续期，并触发 `--install-cert` 更新你的文件。

你也可以手动测试续期：

```
~/.acme.sh/acme.sh --renew -d aitaking.com --force
```


## ✅ 第二步：完整部署流程（宿主机 + systemd）


### 1. 确保证书已正确安装

你之前已用 `acme.sh` 安装证书到：

```
/root/derp-server/certs/fullchain.pem
/root/derp-server/certs/privkey.pem
```

验证：

```
ls -l /root/derp-server/certs/
openssl x509 -in /root/derp-server/certs/fullchain.pem -noout -subject
# 应输出: subject=CN = *.aitaking.com
```

---

### 2. 下载官方预编译 `derper` 二进制（ARM64）

```
cd /root/derp-server

# 获取最新版本号
VERSION=$(curl -s https://api.github.com/repos/tailscale/tailscale/releases/latest | grep '"tag_name"' | cut -d '"' -f 4)
echo "Installing Tailscale DERP version: $VERSION"

# 下载 ARM64 版本（Orange Pi Zero 3 是 arm64/aarch64）
wget -O derper.tgz "https://github.com/tailscale/tailscale/releases/download/${VERSION}/derper_${VERSION#v}_linux_arm64.tgz"

# 解压（会生成 ./derper）
tar -xzf derper.tgz

# 验证
./derper --help
```

> ✅ 不需要 Go，不需要编译！

---

### 3. 创建 systemd 服务（开机自启 + 自动重启）

```
cat > /etc/systemd/system/derper.service <<EOF
[Unit]
Description=DERP Server for Tailscale
After=network.target

[Service]
User=root
WorkingDirectory=/root/derp-server
ExecStart=/root/derp-server/derper \\
  --hostname=derp.aitaking.com \\
  --stun \\
  --http-port=33445 \\
  --tls-cert-path=/root/derp-server/certs/fullchain.pem \\
  --tls-key-path=/root/derp-server/certs/privkey.pem \\
  --a=0.0.0.0
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

> 🔒 参数说明：
> 
> - `--hostname=derp.aitaking.com`：必须匹配证书中的域名（`*.aitaking.com` 覆盖它）
> - `--http-port=33445`：实际是 HTTPS 端口（可自定义）
> - `--stun`：启用 STUN 服务（UDP 3478）
> - `--a=0.0.0.0`：允许所有源 IP 连接（生产环境可用）

---

### 4. 启动服务

```
# 重载 systemd 配置
systemctl daemon-reexec

# 启用开机自启 + 立即启动
systemctl enable --now derper

# 查看实时日志
journalctl -u derper -f
```

✅ 成功标志（日志中出现）：

```
listening on :33445
STUN server listening on :3478
```

---

### 5. 开放防火墙端口

```
# 如果使用 ufw
ufw allow 33445/tcp   # DERP (HTTPS/WSS)
ufw allow 3478/udp    # STUN

# 如果使用 iptables 或云服务器安全组，请确保开放：
# TCP 33445
# UDP 3478
```

---

### 6. 验证外部可访问性（可选）

从另一台机器测试：

```
# 测试 TCP 端口
telnet derp.aitaking.com 33445

# 或用 openssl 测试 TLS
openssl s_client -connect derp.aitaking.com:33445 -servername derp.aitaking.com
```

应能成功建立 TLS 连接。

---

## 🔄 证书自动续期（已配置好！）

你之前用 `acme.sh --install-cert` 安装证书时，**`acme.sh` 已自动设置 cron 任务**，每 60 天续期一次，并自动更新：

```
/root/derp-server/certs/fullchain.pem
/root/derp-server/certs/privkey.pem
```

但 `derper` **不会自动加载新证书**，所以需要**重启服务**。

### 添加自动重启脚本（推荐）

```
# 创建续期后钩子
cat > ~/.acme.sh/aitaking.com_ecc/renew-hook.sh <<'EOF'
#!/bin/bash
systemctl restart derper
logger "DERP service restarted after certificate renewal"
EOF

chmod +x ~/.acme.sh/aitaking.com_ecc/renew-hook.sh
```

然后编辑 `acme.sh` 的 cron 任务：


```
crontab -e
```

找到类似这行：

```
0 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

**改成**：

```
0 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" --renew-hook "/root/.acme.sh/aitaking.com_ecc/renew-hook.sh" > /dev/null
```

> ✅ 这样每次证书更新后，`derper` 会自动重启加载新证书。

---

## 📌 最终目录结构

```
/root/derp-server/
├── derper                     # 二进制程序
├── certs/
│   ├── fullchain.pem          # 证书（含中间 CA）
│   └── privkey.pem            # 私钥
└── (无 docker-compose.yml)
```

---

## ✅ 总结：你现在拥有的是一个

- **轻量级**：仅一个二进制 + 证书
- **自动续期**：`acme.sh` + 钩子脚本
- **开机自启**：systemd 管理
- **安全可靠**：官方原生 `derper`，ARM64 优化















