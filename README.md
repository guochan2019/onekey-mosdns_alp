# onekey-mosdns_alp

一键在 **Alpine Linux LXC** 上部署 [mosdns](https://github.com/IrineSistiana/mosdns) v5 DNS 转发器，实现：

- **国内域名分流** → 阿里公共 DNS（默认 223.5.5.5/223.6.6.6，部署时可交互修改）
- **国外域名分流** → tailnet VPS dnsmasq（部署时交互必填 100.x 上游，隐私不写死仓库）
- **广告屏蔽** → v2ray 广告规则 + 自定义 blocklist
- **GEO IP 数据** → 从 geoip.dat 解包国内 CIDR 至 ip_set，支持 IP 级匹配
- **DNS 缓存** → 内存缓存 + lazy cache + 磁盘持久化
- **PTR 反查本地裁决** → 内网反查不出网
- **自动更新** → 每周更新域名规则 + GEO 数据 + mosdns 自身版本

> 功能与 [onekey-mosdns](https://github.com/guochan2019/onekey-mosdns)（Debian 直装版）完全一致（config.yaml / GEO 解包 / 分流逻辑字节级相同），平台层适配 **apk + OpenRC + busybox crond**。部署目录相同（`/opt/mosdns`），Linux Gate 上的现成配置可直接迁移。

---

## 快速开始

> ⚠️ 需要 root 权限。适用 Alpine Linux（OpenRC）。官方 release 二进制为纯静态编译（Dockerfile `CGO_ENABLED=0`），musl 直接兼容，无需 gcompat。

```bash
# 方式一：一键直达（推荐）
sh <(wget -qO- https://raw.githubusercontent.com/guochan2019/onekey-mosdns_alp/main/onekey-mosdns_alp.sh)

# 方式二：gh CLI
gh repo clone guochan2019/onekey-mosdns_alp && cd onekey-mosdns_alp
chmod +x onekey-mosdns_alp.sh && ./onekey-mosdns_alp.sh
```

---

## 使用方式

运行脚本后显示菜单：

```
========================================
  mosdns 一键安装/升级/卸载脚本 (Alpine)
  https://github.com/IrineSistiana/mosdns
========================================

[INFO] 检测到 mosdns v5.3.4 已安装

请选择操作：
  1. 安装 / 升级 mosdns
  2. 卸载 mosdns
  0. 退出
```

| 选项 | 功能 |
|------|------|
| **1** | 未安装 → 9 步完整安装；已安装 → 更新 GEO + 检测升级 |
| **2** | 卸载：停止服务、删除全部文件、清除 crontab、恢复 `/etc/resolv.conf` → 223.5.5.5 |
| **0** | 退出 |

## 安装流程

| 步骤 | 说明 |
|------|------|
| 1/9 | 安装依赖（apk: wget、unzip、curl、bind-tools、python3） |
| 2/9 | 检查 resolv.conf 符号链接（Alpine 无 systemd-resolved，无需停用） |
| 3/9 | 下载并安装最新版 mosdns（官方 release 二进制，静态） |
| 4/9 | 安装 GEO 数据解包工具（内嵌 Python 脚本） |
| 5/9 | 创建 `/opt/mosdns/{rule,}` 和日志目录 |
| 6/9 | 下载 GEO 数据（geoip.dat + geosite.dat），解包全部规则 |
| 7/9 | 交互输入本地/远程 DNS 上游 → 写入 config.yaml |
| 8/9 | 创建 OpenRC 服务（supervise-daemon 自动重启） |
| 9/9 | 启动 mosdns，验证解析 |
| 可选 | 设置本机 DNS → 127.0.0.1 |

安装完成后自动进行三项验证：
- `www.baidu.com` → 国内 DNS 解析
- `www.google.com` → 国外 DNS 解析
- `doubleclick.net` → 广告屏蔽

---

## 目录结构

```
/opt/mosdns/
├── config.yaml                  # mosdns 配置文件
├── cache.dump                   # DNS 缓存持久化文件（自动生成）
├── geoip-unpack.py              # GEOIP 数据解包脚本（从 .dat 提取 CIDR）
├── rule/
│   ├── geosite_cn.txt           # 国内域名列表
│   ├── geosite_geolocation-!cn.txt  # 国外域名列表
│   ├── geosite_category-ads-all.txt # 广告/跟踪域名列表
│   ├── geoip_cn.txt             # 国内 IP 段列表（CIDR，从 geoip.dat 解包）
│   ├── whitelist.txt            # 白名单域名（走国内 DNS，可选）
│   ├── blocklist.txt            # 自定义拦截域名（可选）
│   └── hosts.txt                # 自定义 hosts 映射（可选）
└── update-mosdns.sh             # 自动更新脚本（crontab 每周一执行）

/usr/share/v2ray/                # 全局 GEO 共享目录
├── geoip.dat                    # IP 段数据库（二进制）
└── geosite.dat                  # 域名数据库（二进制）

/var/log/mosdns/
├── mosdns.log                   # mosdns 运行日志
└── update-mosdns.log            # 更新脚本日志
```

---

## 配置说明

### DNS 上游（首次部署交互输入，可预设跳过）

| 项 | 输入 | 默认/必填 |
|----|------|-----------|
| 本地 DNS（forward_local） | 空格分隔纯 IP | 默认 `223.5.5.5 223.6.6.6`（阿里公共无地域特征），回车即用 |
| 远程 DNS（forward_remote） | 空格分隔纯 IP | **必填**（tailnet VPS dnsmasq 的 100.x，脚本自动加 `udp://` 前缀）；🔴 不写死进仓库 |

> 脚本顶部预设 `LOCAL_DNS_IPS` / `REMOTE_DNS_IPS` 后跳过提问（空 = 交互输入）。

### 处理流程

```
客户端请求 → mosdns :53
  ├─ 白名单域名 → 国内 DNS（优先级最高）
  ├─ 广告域名/blocklist → reject (NXDOMAIN)
  ├─ qtype 65 (HTTPS) → reject（减少 QUIC 泄漏）
  ├─ 查询缓存 → 命中则直接返回
  ├─ PTR 反查 (qtype 12) → 本地 DNS（内网反查不出网）
  ├─ 国内域名 → 国内 DNS
  ├─ 国外域名 → 远程 DNS（prefer_ipv4）
  └─ 其余 → fallback 双检
       ├─ primary: 国内 DNS → 结果非国内 IP 则丢弃
       └─ secondary: 远程 DNS（500ms 超时）
```

---

## 服务管理（OpenRC）

```bash
rc-service mosdns status       # 查看状态
rc-service mosdns restart      # 重启
rc-service mosdns stop         # 停止
tail -f /var/log/mosdns/mosdns.log   # 实时日志（config.yaml 已配置 file 落盘）
```

> 服务由 `supervise-daemon` 托管：进程异常退出自动拉起（对齐 systemd `Restart=on-failure`）。若与 tailscale 同机部署（remote 上游走 tailnet 100.x），先执行 `rc-update add tailscaled default` 再装 mosdns，确保依赖顺序。

### 升级 / 卸载

再次运行脚本选择对应选项即可：

```bash
sh onekey-mosdns_alp.sh
# 选 1 → 升级；选 2 → 卸载
```

---

## API 接口

mosdns 在本机 9091 端口提供 HTTP API：

```bash
# 清空缓存
curl http://127.0.0.1:9091/flush

# 下载缓存快照
curl http://127.0.0.1:9091/dump -o cache.dump
```

---

## 自动更新

脚本自动启用 busybox crond（Alpine 默认未启动），crontab 每周一 03:00 执行 `/opt/mosdns/update-mosdns.sh`，完成两件事：

1. **更新 GEO 数据** — 重下 `geoip.dat` + `geosite.dat` 至 `/usr/share/v2ray/`，解包全部域名规则 + geoip CIDR
2. **升级 mosdns** — 检测 GitHub 最新 release，版本不一致时自动下载替换

更新完成后自动重启 mosdns 使配置生效。

---

## 与 Debian 直装版差异

| 项 | Debian 版 | Alpine 版 |
|----|-----------|-----------|
| 依赖安装 | `apt install wget unzip curl dnsutils python3` | `apk add wget unzip curl bind-tools python3 ca-certificates`（dnsutils→bind-tools） |
| :53 冲突处理 | 停用 systemd-resolved | 无此服务，仅处理 resolv.conf 符号链接 |
| 服务管理 | systemd unit | OpenRC init.d + supervise-daemon（等价自愈） |
| 自动更新触发 | crontab（cron 随系统） | 自动启用 busybox crond（rc-update add crond） |
| 脚本语言 | bash | POSIX sh（Alpine 无 bash，交互 read -p 已改 printf+read） |
| 部署目录 | `/opt/mosdns` + `/usr/local/bin` + `/usr/share/v2ray` | **相同**（config/rule 可直接迁移） |

---

## 许可证

mosdns 上游遵循 GPL-3.0。本脚本未附带独立 LICENSE。
