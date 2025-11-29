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
## 步骤七：启动 DERP（和之前一样）

docker-compose.yml
如果不能拉取到镜像，可以参考下面的自编译创建Dockerfile

```
version: '3'
services:
  derper:
    image: tailscale/derper:latest
    container_name: derper
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./certs:/certs:ro
    command:
      - "--hostname=derp.aitaking.com"
      - "--stun"
      - "--http-port=33445"
      - "--tls-cert-path=/certs/fullchain.pem"
      - "--tls-key-path=/certs/privkey.pem"
      - "--a=0.0.0.0"
```

启动：
```
cd ~/derp-server
docker-compose up -d
```
## 小贴士

- 如果你以后换服务器，只需重新运行 `acme.sh --issue` 即可。
- 阿里云 AccessKey 建议只给 DNS 权限，用完可禁用。
- 证书文件路径固定，DERP 无需重启（但建议重启以加载新证书，可通过 cron 实现）。


### 方案二：自己构建 Docker 镜像（如果你坚持用 Docker）

1. 安装 Go（≥1.21）和 Git
```
apt update && apt install -y git golang
```
```
go version
```
如果你的系统 Go 版本太低（如 Ubuntu 22.04 默认是 1.18），请先升级：

```
# 下载 Go 1.23（当前最新稳定版）
wget https://go.dev/dl/go1.23.4.linux-arm64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.23.4.linux-arm64.tar.gz

# 添加到 PATH（临时）
export PATH=$PATH:/usr/local/go/bin

# 或永久写入 ~/.bashrc
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### 下载官方 release 源码（非 main 分支！）

```
cd /tmp
LATEST_TAG=$(curl -s https://api.github.com/repos/tailscale/tailscale/releases/latest | grep '"tag_name"' | cut -d '"' -f 4)
git clone --depth=1 --branch "$LATEST_TAG" https://github.com/tailscale/tailscale.git
cd tailscale
```

### 构建 `derper` 二进制（用于放入镜像）

```
# 构建 Linux ARM64 版本
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o derper ./cmd/derper
```

验证：
```
file derper
# 应显示: ELF 64-bit LSB executable, ARM aarch64
```

### 准备 Docker 构建目录

```
mkdir -p /root/derp-server/build
cp derper /root/derp-server/build/
cp -r /root/derp-server/certs /root/derp-server/build/
```

### 创建 `Dockerfile`
```
cat > /root/derp-server/build/Dockerfile <<'EOF'
FROM alpine:latest
RUN apk add --no-cache ca-certificates tzdata
WORKDIR /app
COPY derper /app/
COPY certs/ /certs/
EXPOSE 33445
USER nobody
CMD ["./derper", \
  "--hostname=derp.aitaking.com", \
  "--stun", \
  "--http-port=33445", \
  "--tls-cert-path=/certs/fullchain.pem", \
  "--tls-key-path=/certs/privkey.pem", \
  "--a=0.0.0.0"]
EOF
```

### 构建 Docker 镜像

```
cd /root/derp-server/build
docker build -t my-derper:latest .
```

成功后你会看到：

```
Successfully built xxxxxxxx
Successfully tagged my-derper:latest
```
### 修改 `docker-compose.yml`


现在你可以用自建镜像了：

```
# /root/derp-server/docker-compose.yml
version: '3.8'
services:
  derper:
    image: my-derper:latest    # ← 使用本地镜像
    container_name: derper
    restart: unless-stopped
    network_mode: host
    # 不需要 volumes，因为证书已打包进镜像
```

💡 由于证书已 COPY 进镜像，**无需挂载 volumes**（更安全，避免权限问题）。

### 启动服务

```
cd /root/derp-server
docker-compose up -d
```
查看日志：

```
docker logs derper
```
应看到：
```
listening on :33445
STUN server listening on :3478
```


