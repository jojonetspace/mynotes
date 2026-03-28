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

## 完善后的 Tailscale 一键安装脚本

```bash
#!/bin/bash
# Orange Pi Zero 3 Tailscale 一键安装配置脚本
# 功能：安装 Tailscale，配置子网路由、出口节点、NAT 转发和防火墙规则
# 子网路由：10.201.0.0/16,192.168.20.0/24,192.168.48.0/24,192.168.6.0/24,192.168.38.0/24
# 创建时间：2026-03-28
# 版本：v2.0 - 集成 NAT 和防火墙配置

set -e

echo "=========================================="
echo "Orange Pi Zero 3 Tailscale 一键安装配置脚本"
echo "=========================================="

# 定义变量
SUBNET_ROUTES="10.201.0.0/16,192.168.20.0/24,192.168.48.0/24,192.168.6.0/24,192.168.38.0/24"
TAILSCALE_INSTALL_URL="https://tailscale.com/install.sh"
SERVICE_NAME="tailscaled"
SCRIPT_VERSION="v2.0"
DEFAULT_HOSTNAME="op-jst"

# 检查是否以 root 运行
if [ "$EUID" -ne 0 ]; then
    echo "❌ 请使用 root 权限运行此脚本：sudo bash $0"
    exit 1
fi

# 记录日志函数
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a /tmp/tailscale_install.log
}

# 错误处理函数
handle_error() {
    log_message "❌ 错误：$1"
    echo "❌ 安装过程中发生错误：$1"
    echo "📄 查看日志：tail -f /tmp/tailscale_install.log"
    exit 1
}

# 1. 安装 Tailscale
log_message "步骤 1/5：安装 Tailscale..."
if command -v tailscale &> /dev/null; then
    echo "   ✅ Tailscale 已安装，版本：$(tailscale version | head -1)"
else
    echo "   🔄 正在安装 Tailscale..."
    curl -fsSL $TAILSCALE_INSTALL_URL 2>&1 | tee -a /tmp/tailscale_install.log | sh
    if [ ${PIPESTATUS[0]} -eq 0 ]; then
        echo "   ✅ Tailscale 安装成功"
    else
        handle_error "Tailscale 安装失败，请检查网络连接"
    fi
fi

# 2. 配置系统网络参数
log_message "步骤 2/5：配置系统网络参数..."
echo "   🔄 启用 IP 转发..."

# 启用 IPv4 转发
if ! grep -q "^net.ipv4.ip_forward=1" /etc/sysctl.conf; then
    echo "net.ipv4.ip_forward=1" | tee -a /etc/sysctl.conf
fi

# 启用 IPv6 转发
if ! grep -q "^net.ipv6.conf.all.forwarding=1" /etc/sysctl.conf; then
    echo "net.ipv6.conf.all.forwarding=1" | tee -a /etc/sysctl.conf
fi

# 应用配置
sysctl -p 2>&1 | tee -a /tmp/tailscale_install.log
echo 1 > /proc/sys/net/ipv4/ip_forward
echo "   ✅ IP 转发已配置"

# 3. 配置网络接口和防火墙规则
log_message "步骤 3/5：配置网络接口和防火墙规则..."

# 获取默认路由的网卡
PUBLIC_INTERFACE=$(ip route show default 2>/dev/null | awk '{print $5}' | head -1)
if [ -z "$PUBLIC_INTERFACE" ]; then
    # 如果没有默认路由，尝试使用第一个非lo的网卡
    PUBLIC_INTERFACE=$(ip link show | grep -E "^[0-9]+:" | grep -v lo | awk -F: '{print $2}' | tr -d ' ' | head -1)
fi

if [ -n "$PUBLIC_INTERFACE" ]; then
    echo "   🔧 检测到公网网卡: $PUBLIC_INTERFACE"
    
    # 添加 MASQUERADE 规则
    echo "   🔧 配置 NAT 规则..."
    iptables -t nat -C POSTROUTING -o $PUBLIC_INTERFACE -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o $PUBLIC_INTERFACE -j MASQUERADE
    iptables -t nat -C POSTROUTING -o tailscale0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o tailscale0 -j MASQUERADE
    
    # 配置 FORWARD 链规则
    echo "   🔧 配置防火墙转发规则..."
    iptables -D FORWARD -s 100.64.0.0/10 -j DROP 2>/dev/null || true
    iptables -A FORWARD -i tailscale0 -j ACCEPT
    iptables -A FORWARD -o tailscale0 -j ACCEPT
    
    # 保存 iptables 规则
    echo "   💾 保存防火墙规则..."
    mkdir -p /etc/iptables
    iptables-save > /etc/iptables/rules.v4
    echo "   ✅ 网络规则配置完成"
else
    echo "   ⚠️  警告：未检测到网络接口，跳过防火墙规则配置"
fi

# 4. 启动 Tailscale 服务
log_message "步骤 4/5：启动 Tailscale 服务..."
systemctl enable --now $SERVICE_NAME 2>&1 | tee -a /tmp/tailscale_install.log
systemctl restart $SERVICE_NAME 2>&1 | tee -a /tmp/tailscale_install.log
sleep 3

echo "   ✅ Tailscale 服务已启动"

# 5. 配置 Tailscale
log_message "步骤 5/5：配置 Tailscale 网络..."
echo ""
echo "📋 配置详情："
echo "   • 设备名称: $DEFAULT_HOSTNAME"
echo "   • 子网路由: $SUBNET_ROUTES"
echo "   • 出口节点: 已启用"
echo ""

# 检查是否已登录
if tailscale status 2>/dev/null | grep -q "Logged in"; then
    echo "   ✅ Tailscale 已登录，重新配置路由和出口节点..."
    tailscale up --hostname=$DEFAULT_HOSTNAME --advertise-routes=$SUBNET_ROUTES --advertise-exit-node --reset
else
    echo "📝 请选择认证方式："
    echo "   1) 使用浏览器认证（推荐）"
    echo "   2) 使用认证密钥（Auth Key）"
    echo "   3) 仅生成命令，手动执行"
    echo ""
    read -p "请选择 [1-3]: " auth_choice

    case $auth_choice in
        1)
            echo ""
            echo "🔗 正在生成认证链接..."
            echo "   请复制下面的链接到浏览器中打开并登录："
            echo "   -------------------------------------------------"
            tailscale up --hostname=$DEFAULT_HOSTNAME --advertise-routes=$SUBNET_ROUTES --advertise-exit-node --reset
            echo "   -------------------------------------------------"
            ;;
        2)
            read -p "请输入 Tailscale 认证密钥: " auth_key
            if [ -n "$auth_key" ]; then
                echo "🔄 使用认证密钥登录..."
                tailscale up --hostname=$DEFAULT_HOSTNAME --authkey=$auth_key --advertise-routes=$SUBNET_ROUTES --advertise-exit-node --reset
                echo "✅ 认证完成"
            else
                echo "❌ 认证密钥不能为空"
                exit 1
            fi
            ;;
        3)
            echo ""
            echo "📋 请手动执行以下命令："
            echo "   tailscale up --hostname=$DEFAULT_HOSTNAME --advertise-routes=$SUBNET_ROUTES --advertise-exit-node --reset"
            echo ""
            echo "   执行后，复制输出的链接到浏览器登录。"
            ;;
        *)
            echo "❌ 无效选择"
            exit 1
            ;;
    esac
fi

# 安装 iptables-persistent 以保存规则
echo ""
echo "🔧 安装 iptables-persistent 以持久化防火墙规则..."
if ! command -v netfilter-persistent &> /dev/null; then
    apt-get update 2>&1 | tee -a /tmp/tailscale_install.log
    apt-get install -y iptables-persistent 2>&1 | tee -a /tmp/tailscale_install.log
    echo "   ✅ iptables-persistent 已安装"
    
    # 保存当前规则
    if command -v netfilter-persistent &> /dev/null; then
        netfilter-persistent save 2>&1 | tee -a /tmp/tailscale_install.log
        echo "   ✅ 防火墙规则已持久化"
    fi
else
    echo "   ✅ iptables-persistent 已安装"
    netfilter-persistent save 2>&1 | tee -a /tmp/tailscale_install.log
fi

# 6. 验证安装
echo ""
echo "✅ 安装配置完成！"
echo "=========================================="
echo "📊 验证配置："
echo "   1. 查看 Tailscale 状态: tailscale status"
echo "   2. 查看分配的 IP: tailscale ip"
echo "   3. 查看子网路由: tailscale status --json | grep -i route"
echo "   4. 查看出口节点状态: tailscale exit-node list"
echo ""
echo "⚠️  重要后续步骤："
echo "   1. 登录 Tailscale 管理后台 (https://login.tailscale.com/admin)"
echo "   2. 找到您的设备 ($DEFAULT_HOSTNAME)"
echo "   3. 点击 'Edit route settings' 批准子网路由"
echo "   4. 点击 'Edit exit node' 启用出口节点"
echo ""
echo "🔄 开机自启："
echo "   • Tailscale 服务: systemctl enable tailscaled"
echo "   • 防火墙规则: 已通过 iptables-persistent 持久化"
echo ""
echo "🔧 管理命令："
echo "   • 重启服务: systemctl restart tailscaled"
echo "   • 查看日志: journalctl -u tailscaled -f"
echo "   • 断开连接: tailscale down"
echo "   • 重新连接: tailscale up"
echo "   • 修改配置: nano /etc/sysctl.conf"
echo ""
echo "📄 安装日志: /tmp/tailscale_install.log"
log_message "Tailscale 安装配置完成"
```

## 与原安装脚本的主要改进：

1. **自动检测网络接口**：自动获取默认网卡名称
    
2. **配置 NAT 规则**：添加必要的 MASQUERADE 规则
    
3. **配置防火墙规则**：允许 tailscale0 接口的转发
    
4. **持久化防火墙规则**：安装并配置 iptables-persistent
    
5. **默认设备名称为 op-jst**：自动设置设备名称为 op-jst
    
6. **更完善的错误处理**：添加日志记录和错误处理
    
7. **自动重启服务**：确保配置生效
    
8. **检查已登录状态**：如果已登录则跳过认证步骤
    

## 使用方法：

```bash
# 1. 下载脚本
curl -O https://your-domain.com/orange-pi-tailscale-install.sh
# 或手动创建脚本
nano install_tailscale_complete.sh

# 2. 赋予执行权限
chmod +x install_tailscale_complete.sh

# 3. 运行脚本
sudo ./install_tailscale_complete.sh
```

## 如果已经执行了原安装脚本：

如果您已经执行了原安装脚本，但缺少防火墙规则，可以直接执行以下命令来添加必要的规则：

```bash
#!/bin/bash
# 补充配置脚本 - 用于已安装 Tailscale 但缺少防火墙规则的情况

# 开启 IP 转发
echo 1 > /proc/sys/net/ipv4/ip_forward
echo 'net.ipv4.ip_forward = 1' | tee -a /etc/sysctl.conf
sysctl -p

# 获取公网网卡
PUBLIC_INTERFACE=$(ip route show default | awk '{print $5}')
echo "公网网卡: $PUBLIC_INTERFACE"

# 添加 NAT 规则
iptables -t nat -A POSTROUTING -o $PUBLIC_INTERFACE -j MASQUERADE
iptables -t nat -A POSTROUTING -o tailscale0 -j MASQUERADE

# 修复防火墙规则
iptables -D FORWARD -s 100.64.0.0/10 -j DROP 2>/dev/null || true
iptables -A FORWARD -i tailscale0 -j ACCEPT
iptables -A FORWARD -o tailscale0 -j ACCEPT

# 保存规则
apt-get install -y iptables-persistent
netfilter-persistent save

echo "补充配置完成！"
```

这个完善的脚本已经包含了所有必要的配置步骤，运行一次即可完成 Tailscale 的安装、配置和防火墙规则设置。

sudo tailscale up --accept-routes=true --advertise-routes=10.201.0.0/16,192.168.20.0/24,192.168.48.0/24,192.168.6.0/24,192.168.38.0/24 --advertise-exit-node --reset


![[Pasted image 20260328141422.png]]

公司网络有两个设备，一个windows的lizh-tb16，一个unbuntu的op-jst都安装了tailscale。家里有一个同样是unbuntu的orangepizero3安装了tailscale,家里的路由器配置了静态路由见图。我还在阿里云安装了一个dep的服务用以家庭和公司的组网。
derp服务check地址：https://8.153.165.65:33445/
![[Pasted image 20260328140420.png]]