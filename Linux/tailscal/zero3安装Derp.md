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

2. 克隆源码并构建

```
cd /tmp
git clone https://github.com/tailscale/tailscale.git
cd tailscale

# 构建 derper
go build -o derper ./cmd/derper
```







