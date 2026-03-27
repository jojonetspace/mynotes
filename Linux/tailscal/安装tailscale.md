家庭网络：
在192.168.5.3上安装tailscale
在主路由的静态路由设置：
100.64.0.0	255.192.0.0	192.168.5.3		LAN	
192.168.20.0	255.255.255.0	192.168.5.3		LAN	
192.168.6.0	255.255.255.0	192.168.5.3		LAN	
192.168.48.0	255.255.255.0	192.168.5.3		LAN



公司：
1）在windows设备管理器找到对应的网卡，然后在高级中找到本地地址，将mac地址修改为linux机器的地址
2）在windows浏览器上完成钉钉认证

tailscale 安装脚本：
```
# 使用cat命令直接创建脚本文件
cat > setup_tailscale.sh << 'EOF'
#!/bin/bash

echo "=== Tailscale安装配置脚本（Ubuntu 22.04专用）==="
echo "正在安装Tailscale..."

# 1. 安装Tailscale
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
sudo apt update -q
sudo apt install -y tailscale

echo "启用IP转发..."
# 2. 启用IP转发
sudo tee /etc/sysctl.d/99-tailscale.conf << INNER_EOF
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
INNER_EOF
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

echo "配置防火墙..."
# 3. 配置防火墙
sudo ufw allow in on tailscale0
sudo ufw allow 41641/udp
sudo ufw --force enable > /dev/null 2>&1

echo ""
echo "=== 重要！请手动执行最后一步 ==="
echo "运行以下命令启动Tailscale并认证："
echo "sudo tailscale up --advertise-routes=10.201.0.0/16,192.168.20.0/24,192.168.48.0/24,192.168.6.0/24,192.168.38.0/24 --advertise-exit-node --reset"
echo ""
echo "认证后，请务必访问："
echo "https://login.tailscale.com/admin"
echo "1. 找到orangepizero3设备"
echo "2. 批准所有子网路由（5个网段）"
echo "3. 启用'Use as exit node'"
echo "4. 点击Save"
echo "=== 脚本第一部分完成 ==="
EOF
```

**然后执行：**

```bash
# 给脚本执行权限
chmod +x setup_tailscale.sh

# 查看文件是否创建成功
ls -la setup_tailscale.sh

# 运行脚本
./setup_tailscale.sh
```

