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
mkdir -p ~/derp-server/certs
```