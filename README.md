# ECH Workers 客户端


## 📋 目录


- [软路由部署](#软路由部署)
- [系统要求](#系统要求)
- [故障排除](#故障排除)
- [技术文档](#技术文档)



## 🛣️ 软路由部署
### 图形化推荐
https://github.com/SunshineList/luci-app-ech-workers
### OpenWrt 部署
### 一键脚本
```bash

wget https://raw.githubusercontent.com/Syntheticlight/ech-wk-armv7/refs/heads/main/softrouter.sh
chmod +x softrouter.sh
./softrouter.sh
```
```bash

#后续使用只需要这一行
./softrouter.sh
```
#### 1. 上传文件

```bash
# 通过 SCP 上传
scp ech-workers root@192.168.1.1:/usr/bin/

# 或通过 WinSCP、FileZilla 等工具上传
```

#### 2. 设置执行权限

```bash
ssh root@192.168.1.1
chmod +x /usr/bin/ech-workers
```

#### 3. 创建启动脚本

创建 `/etc/init.d/ech-workers`：

```bash
#!/bin/sh /etc/rc.common

START=99
STOP=10
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/ech-workers \
        -f your-worker.workers.dev:443 \
        -l 0.0.0.0:30001 \
        -token your-token \
        -routing bypass_cn
    procd_set_param respawn
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

设置权限：
```bash
chmod +x /etc/init.d/ech-workers
```

#### 4. 启用并启动服务

```bash
/etc/init.d/ech-workers enable
/etc/init.d/ech-workers start
```

#### 5. 查看服务状态

```bash
/etc/init.d/ech-workers status
logread | grep ech-workers
```

#### 6. 配置 OpenWrt 代理

在 PassWall、OpenClash 等插件中配置：
- 代理类型: SOCKS5
- 服务器: `软路由的IP`
- 端口: `30001`

### iKuai 软路由部署

#### 1. 上传文件

通过 iKuai 的 Web 管理界面或 SSH 上传 `ech-workers` 到 `/bin/` 目录。

#### 2. 创建启动脚本

创建 `/etc/init.d/ech-workers.sh`：

```bash
#!/bin/sh
/bin/ech-workers -f your-worker.workers.dev:443 -l 127.0.0.1:30001 -routing bypass_cn &
```

设置权限：
```bash
chmod +x /etc/init.d/ech-workers.sh
```

#### 3. 添加到开机启动

编辑 `/etc/rc.local`，添加：
```bash
/etc/init.d/ech-workers.sh
```
### Docker部署
   
参数说明：

```
--network host #网络类型一般推荐直接host 
  -e ARG_F="" #填写你的workers域名和端口 
  -e ARG_ECH="cloudflare-ech.com" #ech查询域名，一般保持默认 
  -e ARG_TOKEN="" #你设置的token
  -e ARG_IP="visa.com" #优选IP或域名
  -e ARG_L="0.0.0.0:30000" #Socks5服务器的IP和端口，0.0.0.0为全局监听
  -e ARG_ROUTING="global" #分流模式，global=全局代理 bypass_cn=绕过大陆
```

docker运行命令模板，按照上面的说明填写，然后复制到终端运行：

```
docker run -d \
  --name cfech \
  --restart always \
  --network host \
  -e ARG_F="" \
  -e ARG_ECH="cloudflare-ech.com" \
  -e ARG_TOKEN="" \
  -e ARG_IP="visa.com" \
  -e ARG_L="0.0.0.0:30000" \
  -e ARG_ROUTING="global" \
  cirnosalt/ech-workers-docker:latest
```
   
### 软路由配置建议

#### 网络配置

```bash
# 监听所有网络接口（推荐）
./ech-workers -f your-worker.workers.dev:443 -l 0.0.0.0:30001 -routing bypass_cn

# 或仅监听内网接口
./ech-workers -f your-worker.workers.dev:443 -l 192.168.1.1:30001 -routing bypass_cn
```

#### 防火墙规则

确保防火墙允许代理端口：

```bash
# OpenWrt
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-ECH-Workers'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest_port='30001'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall
/etc/init.d/firewall reload
```

#### 性能优化

- 使用 `-ip` 参数指定固定 IP，减少 DNS 查询
- 调整系统资源限制（如文件描述符数量）
- 考虑使用 `systemd` 或 `procd` 管理进程

#### 监控和日志

```bash
# 查看进程状态
ps | grep ech-workers

# 查看日志
logread | grep ech-workers

# 测试连接
curl --socks5 127.0.0.1:30001 http://www.google.com
```

## 📋 系统要求

### 操作系统
- **Windows**: Windows 10+ (Windows 11 完全支持)
- **macOS**: macOS 10.13+
- **Linux**: Ubuntu 18.04+ / Debian 10+ / CentOS 7+ (支持 x86_64 和 ARM64)

### 运行时要求
- **预编译版本**: 无需额外依赖，可直接运行
- **从源码编译**: 
  - Python 3.6+ (仅 GUI 需要)
  - Go 1.23+ (仅编译时需要，ECH 功能需要此版本)

### 网络要求
- 能够访问 GitHub (用于自动下载 IP 列表)
- 能够访问 Cloudflare Workers 服务

## 🔧 故障排除

### IP 列表下载失败

**问题**: 程序无法下载中国 IP 列表

**解决方案**:
1. 检查网络连接，确保能够访问 GitHub
2. 手动下载 IP 列表文件：
   ```bash
   # IPv4 列表
   curl -L -o chn_ip.txt "https://raw.githubusercontent.com/mayaxcn/china-ip-list/refs/heads/master/chn_ip.txt"
   
   # IPv6 列表
   curl -L -o chn_ip_v6.txt "https://raw.githubusercontent.com/mayaxcn/china-ip-list/refs/heads/master/chn_ip_v6.txt"
   ```
3. 将文件放在程序目录下，程序会自动使用

### 找不到 ech-workers

**问题**: GUI 提示找不到 `ech-workers` 可执行文件

**解决方案**:
1. 确保已编译 Go 程序：
   ```bash
   go build -o ech-workers ech-workers.go
   ```
2. 确保 `ech-workers` 与 GUI 在同一目录
3. 检查文件执行权限（Linux/macOS）：
   ```bash
   chmod +x ech-workers
   ```

### Windows 11 系统代理问题

**问题**: Windows 11 系统代理设置失败

**解决方案**:
1. 确保以管理员权限运行程序
2. 检查防火墙设置
3. 程序会自动使用正确的代理格式（`127.0.0.1:端口`）

### Linux 系统代理设置

**问题**: Linux 不支持自动设置系统代理

**解决方案**:
- 在系统设置中配置 SOCKS5 代理为 `127.0.0.1:端口`
- 或使用环境变量：
  ```bash
  export ALL_PROXY=socks5://127.0.0.1:端口
  ```

### bad CPU type in executable (macOS)

**问题**: 在 macOS 上运行时报错 "bad CPU type"

**解决方案**:
- Intel Mac 请下载 `darwin-amd64` 版本
- Apple Silicon Mac 请下载 `darwin-arm64` 版本

### PyQt5 安装问题

**问题**: GUI 无法启动，提示 PyQt5 未安装

**解决方案**:
```bash
# macOS
pip3 install --user PyQt5

# Windows
pip install PyQt5

# Linux (Debian/Ubuntu)
sudo apt install python3-pyqt5
```

### 软路由重启后服务未启动

**问题**: 软路由重启后代理服务未启动

**解决方案**:
1. 检查启动脚本权限：
   ```bash
   chmod +x /etc/init.d/ech-workers
   ```
2. 确保服务已启用：
   ```bash
   /etc/init.d/ech-workers enable
   ```
3. 检查系统日志：
   ```bash
   logread | grep ech-workers
   ```

## 📚 技术文档

### ECH (Encrypted Client Hello)

ECH 是 TLS 1.3 的扩展功能，用于加密 TLS 握手中的 SNI（服务器名称指示），提供更强的隐私保护。这是本程序的**核心功能**，需要 Go 1.23+ 支持。

### 中国 IP 列表

程序会自动从 [mayaxcn/china-ip-list](https://github.com/mayaxcn/china-ip-list) 下载完整的中国 IP 列表，用于"跳过中国大陆"分流模式。

- **IPv4 列表**: `chn_ip.txt` - 包含约 8000+ 个 IPv4 地址段
- **IPv6 列表**: `chn_ip_v6.txt` - 包含完整的中国 IPv6 地址段

列表文件保存在程序目录，如果文件不存在或为空，程序会自动下载。

### 分流逻辑

分流判断在 Go 核心程序中实现，使用高效的二分查找算法：

1. **域名解析**: 如果目标是域名，先解析为 IP 地址
2. **IP 检查**: 检查所有解析到的 IP（IPv4/IPv6）
3. **范围匹配**: 使用二分查找在 IP 列表中查找匹配
4. **决策**: 根据分流模式决定是否走代理

### 系统代理设置

- **Windows**: 通过注册表设置系统代理，支持 SOCKS5 代理
- **macOS**: 使用 `networksetup` 命令设置所有网络服务的 SOCKS 代理
- **Linux**: 暂不支持自动设置，需要手动配置

### 配置文件

配置文件保存在：
- **Windows**: `%APPDATA%\ECHWorkersClient\config.json`
- **macOS**: `~/Library/Application Support/ECHWorkersClient/config.json`
- **Linux**: `~/.config/ECHWorkersClient/config.json`

## 🤝 致谢

本项目的客户端和 Go 核心程序均基于 [CF_NAT](https://t.me/CF_NAT) 的原始代码开发。

- **原始项目来源**: [CF_NAT - 中转](https://t.me/CF_NAT)
- **Telegram 频道**: [@CF_NAT](https://t.me/CF_NAT)
- **中国 IP 列表**: [mayaxcn/china-ip-list](https://github.com/mayaxcn/china-ip-list)

## 📄 许可证

本项目基于 [CF_NAT](https://t.me/CF_NAT) 的原始代码开发。

## 🌟 Star History

<a href="https://www.star-history.com/#byJoey/ech-wk&type=timeline&logscale&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=byJoey/ech-wk&type=timeline&theme=dark&logscale&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=byJoey/ech-wk&type=timeline&logscale&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=byJoey/ech-wk&type=timeline&logscale&legend=top-left" />
 </picture>
</a>

## 📞 联系方式

- **Telegram 交流群**: https://t.me/+ft-zI76oovgwNmRh
- **GitHub Issues**: [提交问题](https://github.com/byJoey/ech-wk/issues)

## 🙏 贡献

欢迎提交 Issue 和 Pull Request！

---

**注意**: 本项目仅供学习和研究使用，请遵守当地法律法规。
