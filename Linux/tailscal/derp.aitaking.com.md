下面为你整理一份 **清晰、可执行、零遗漏的完整部署步骤清单**，包含：

- ✅ 域名规划（`headscale.aitaking.com` + `derp.aitaking.com`）
- ✅ 阿里云：Headscale 控制平面部署
- ✅ 家庭服务器：DERP + STUN 部署（IPv6）
- ✅ DNS 配置
- ✅ 安全与验证

---

## 🌐 一、域名与 DNS 规划

|服务|域名|记录类型|指向|
|---|---|---|---|
|Headscale（控制面）|`headscale.aitaking.com`|**A 记录**|阿里云公网 IPv4|
|DERP + STUN（数据面）|`derp.aitaking.com`|**AAAA 记录**|家庭公网 IPv6|

> 💡 不要为 `derp.aitaking.com` 设置 A 记录（避免 IPv4 客户端连到低速节点）

---

## ☁️ 二、阿里云服务器：部署 Headscale（控制平面）

### 步骤 1：准备环境（Ubuntu/CentOS 示例）

```
# 安装 Docker 和 Docker Compose
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 步骤 2：创建目录结构

```
mkdir -p ~/headscale/{config,data}
cd ~/headscale
```

### 步骤 3：创建 `docker-compose.yml`

```
# ~/headscale/docker-compose.yml
version: '3'
services:
  headscale:
    image: headscale/headscale:latest
    container_name: headscale
    restart: always
    network_mode: host
    volumes:
      - ./config:/etc/headscale
      - ./data:/var/lib/headscale
    command: headscale serve
```

### 步骤 4：初始化配置文件

```
# 首次运行以生成 config.yaml
docker compose up -d
docker compose down

# 编辑 config.yaml
nano config/config.yaml
```

### 步骤 5：配置 `config/config.yaml`

```
# config/config.yaml
server_url: https://headscale.aitaking.com
listen_addr: "[::]:8080"
metrics_listen_addr: "[::]:9090"

# TLS 由 Caddy 处理（见下一步）
private_key_path: ""
certificate_path: ""

# DERP 配置：指向你的家庭中继
derp:
  server:
    enabled: true
    region_id: 999
    region_code: "home-ipv6"
    region_name: "Home (IPv6, 100Mbps)"
    nodes:
      - name: "1"
        host: derp.aitaking.com
        stun_only: false

# 启用 CLI 管理
cli:
  address: "127.0.0.1:8081"

# 其他建议设置
ip_prefixes:
  - "100.64.0.0/10"
  - "fd7a:115c:a1e0::/48"
```

### 步骤 6：添加 Caddy（自动 HTTPS）

```
# 在 docker-compose.yml 中追加 caddy 服务
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: always
    network_mode: host
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
```

创建 `Caddyfile`：

text

编辑

```
# ~/headscale/Caddyfile
headscale.aitaking.com {
  reverse_proxy localhost:8080
}
```

### 步骤 7：启动服务

bash

编辑

```
docker compose up -d
```

> ✅ 现在访问 `https://headscale.aitaking.com` 应显示 Headscale API（JSON 格式）

---

## 🏠 三、家庭服务器：部署 DERP + STUN（IPv6）

### 步骤 1：确保有公网 IPv6

bash

编辑

```
ip -6 addr show | grep "global"
# 应看到类似 240e:xxx::xxxx 的地址
```

### 步骤 2：开放防火墙（UDP 3478 + TCP 443）

bash

编辑

```
# UFW 示例
sudo ufw allow proto tcp from ::/0 to any port 443
sudo ufw allow proto udp from ::/0 to any port 3478
```

### 步骤 3：启动 DERP 容器

bash

编辑

```
docker run -d \
  --name=derp-home \
  --restart=always \
  --network=host \
  -v /root/derp-certs:/certs \
  tailscale/derp \
  --hostname=derp.aitaking.com \
  --certdir=/certs \
  --stun \
  --verify-clients \
  --control-url=https://headscale.aitaking.com
```

> ✅ Let's Encrypt 会自动申请证书（因 `derp.aitaking.com` 可被公网 IPv6 访问）

---

## 🌍 四、DNS 配置（阿里云云解析）

登录 阿里云 DNS 控制台，为 `aitaking.com` 添加：

|主机记录|记录类型|记录值|TTL|
|---|---|---|---|
|`headscale`|A|`你的阿里云公网IPv4`|600|
|`derp`|AAAA|`你的家庭公网IPv6`|600|

> ⚠️ **不要为 `derp` 添加 A 记录！**

---

## 🔐 五、安全加固

### 1. Headscale 创建用户

bash

编辑

```
# 进入容器
docker exec -it headscale headscale users create myuser

# 生成设备注册命令
docker exec -it headscale headscale --user myuser nodes register --key
```

### 2. 启用 ACL（可选）

创建 `config/acl.json`：

json

编辑

```
{
  "groups": {
    "group:all": ["*"]
  },
  "acls": [
    {"action": "accept", "users": ["group:all"], "ports": ["*:*"]}
  ]
}
```

并在 `config.yaml` 中启用：

yaml

编辑

```
acl_policy_path: /etc/headscale/acl.json
```

---

## 🧪 六、验证部署

### 1. 测试 Headscale

bash

编辑

```
curl -s https://headscale.aitaking.com | jq .
# 应返回 {"message":"Headscale API"}
```

### 2. 测试 DERP 证书

bash

编辑

```
curl -6 https://derp.aitaking.com
# 应返回 DER 错误（非 HTTP），但无连接拒绝
```

### 3. 测试 STUN

bash

编辑

```
# 安装 stunclient（Ubuntu）
sudo apt install stun-client

# 测试
stunclient -6 derp.aitaking.com
# 应返回 Mapped Address（你的家庭 IPv6:port）
```

### 4. 客户端加入网络

bash

编辑

```
tailscale up --login-server https://headscale.aitaking.com
```

### 5. 查看中继状态

bash

编辑

```
tailscale netcheck
```

✅ 期望输出：

json

编辑

```
"Region 999": {
  "Latency": "10ms",
  "Preferred": true,
  "DERP": "derp.aitaking.com:443"
}
```

---

## 📦 七、维护建议

|任务|命令/说明|
|---|---|
|更新 Headscale|`docker compose pull && docker compose up -d`|
|备份 Headscale 数据|定期备份 `~/headscale/data` 目录|
|监控 DERP 日志|`docker logs -f derp-home`|
|自动重启服务|已通过 `--restart=always` 实现|

---

## ✅ 最终架构确认

text

编辑

```
[设备] 
  → 注册: https://headscale.aitaking.com (阿里云 IPv4)
  → 中继: derp.aitaking.com (家庭 IPv6, 100Mbps)
```

- ✅ IPv6 设备：高速直连家庭 DERP
- ✅ IPv4 设备：自动 fallback 官方 DERP（不影响使用）
- ✅ 阿里云仅跑控制面，带宽消耗 < 10MB/天
- ✅ 完全私有，无第三方依赖