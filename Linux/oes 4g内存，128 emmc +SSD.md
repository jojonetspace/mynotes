我的核心诉求
1.tailscale 内网穿透
2.相册 （能够手机同步、同步到网盘、还能支持网盘的图片下载到本地）
3.本地影视库
4.cloudflared隧道
5.最好有nas支持

debian oes img:
https://github.com/ophub/amlogic-s9xxx-armbian/releases/download/Armbian_bookworm_arm64_server_2026.07/Armbian_26.08.0_amlogic_a311d-oes_bookworm_6.12.95_server_2026.07.16.img.gz

- 默认 DHCP 获取 IP
- SSH 端口 22
- 用户 `root`，密码 `1234`（首次登录要改密码）


oes:http://192.168.5.127
casaos:http://192.168.5.127:8080
jellyfin:http://192.168.5.127:8096
Immich:http://192.168.5.127:2283
apikey:WN2CyQsX54sdHYDV8AV7FmH43ajl75LnC1B2ljkXfT0
当前 Immich 的 PostgreSQL 和 Redis 是专用容器，在 Docker 的 `immich_default` 网络内部运行，只有同网络的容器才能访问。

具体配置：

PostgreSQL:

- 数据库名：`immich`
- 用户名：`postgres`
- 密码：`immich_pass`
- 容器名/主机名：`immich_postgres`
- 端口：5432（未暴露到宿主机）

Redis:

- 无密码
- 容器名/主机名：`immich_redis`
- 端口：6379（未暴露到宿主机）
- 没有"数据库名"的概念，Redis 用数字编号（默认 db 0）

alist:http://192.168.5.127:5244


所有 5 项全部完成。总结一下 OES 当前的完整服务栈：

|服务|端口|访问方式|
|---|---|---|
|Samba|445|`\\192.168.5.127\share`（用户 user/user）|
|FileBrowser|8080|`http://192.168.5.127:8080`（admin/admin1234567890）|
|Jellyfin|8096|`http://192.168.5.127:8096`|
|Immich|2283|`http://192.168.5.127:2283`|
|Alist|5244|`http://192.168.5.127:5244`（admin/rtyrtyrty）|
|Cloudflared|-|Cloudflare 隧道|
|Tailscale|-|IP 100.106.117.110|

定时任务：每天凌晨 3 点，rclone 自动将 Immich 照片库同步到百度网盘和阿里云盘的 `/备份/相册/` 目录。

后续待做：

- Immich ML 模块（配置 Windows 台式机）
- 手动测试一次完整同步（`/opt/sync-photos.sh`）