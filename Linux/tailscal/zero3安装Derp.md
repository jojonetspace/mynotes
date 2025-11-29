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

### 2. 确保已经安装go语言

```
go version #必须显示go1.23.4
```
###  一步到位脚本（复制粘贴即可）

```
#!/bin/bash
set -e

# === 1. 确保 Go 已正确安装（≥1.21）===
export PATH=/usr/local/go/bin:$PATH
if ! go version | grep -q "go1\.2[1-9]\|go1\.[3-9]"; then
  echo "❌ Go 1.21+ 未正确安装，请先安装 Go 1.23"
  exit 1
fi

# === 2. 创建工作目录 ===
mkdir -p /root/derp-server/certs
cd /root/derp-server

# === 3. 下载官方 release 源码（v1.90.9）===
echo "📥 下载 Tailscale v1.90.9 源码..."
rm -rf /tmp/tailscale-build
git clone --depth=1 --branch v1.90.9 https://github.com/tailscale/tailscale.git /tmp/tailscale-build

# === 4. 构建 derper（ARM64 静态二进制）===
echo "🔨 构建 derper..."
cd /tmp/tailscale-build
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o /root/derp-server/derper ./cmd/derper

# === 5. 验证构建结果 ===
if [ ! -f /root/derp-server/derper ]; then
  echo "❌ 构建失败：derper 未生成"
  exit 1
fi
chmod +x /root/derp-server/derper
echo "✅ derper 构建成功！"

# === 6. 创建 systemd 服务 ===
cat > /etc/systemd/system/derper.service <<'EOF'
[Unit]
Description=DERP Server for Tailscale
After=network.target

[Service]
User=root
WorkingDirectory=/root/derp-server
ExecStart=/root/derp-server/derper \
  --hostname=derp.aitaking.com \
  --stun \
  --http-port=33445 \
  --tls-cert-path=/root/derp-server/certs/fullchain.pem \
  --tls-key-path=/root/derp-server/certs/privkey.pem \
  --a=0.0.0.0
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# === 7. 启动服务 ===
systemctl daemon-reexec
systemctl enable --now derper

echo "🎉 DERP 服务器已启动！"
echo "📄 日志命令: journalctl -u derper -f"
echo "🔍 成功标志: 'listening on :33445' 和 'STUN server listening on :3478'"
```












