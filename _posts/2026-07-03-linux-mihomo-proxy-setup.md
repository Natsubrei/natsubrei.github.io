---
layout: post
title: "在 Linux 服务器上使用 Mihomo 配置命令行代理"
date: 2026-07-03 12:00:00 +0800
categories: [教程]
tags: [mihomo, proxy, linux, vpn, clash, 代理]
---

这篇笔记记录如何在一台无桌面环境的 Linux 服务器上安装并运行 Mihomo，并通过订阅链接生成配置，实现命令行代理、规则分流、全局模式切换和服务自启动。

## 适用环境

- Linux 服务器或虚拟机
- systemd 服务管理
- x86_64 / amd64 架构
- 已有可用于 Clash/Mihomo 的订阅链接

以下命令默认使用 root 用户执行。如果使用普通用户，请按需加 `sudo`。

## 安装 Mihomo

自动获取最新版本号：

```bash
MIHOMO_VERSION="$(curl -fsSL -o /dev/null -w '%{url_effective}' https://github.com/MetaCubeX/mihomo/releases/latest | sed 's#.*/##')"
echo "$MIHOMO_VERSION"
```

下载对应的 Linux amd64 compatible 版本：

```bash
curl -L "https://github.com/MetaCubeX/mihomo/releases/download/${MIHOMO_VERSION}/mihomo-linux-amd64-compatible-${MIHOMO_VERSION}.gz" \
  -o /tmp/mihomo.gz
```

如果希望固定版本，可以手动指定：

```bash
MIHOMO_VERSION='vX.Y.Z'
```

解压并安装到系统路径：

```bash
gzip -dc /tmp/mihomo.gz > /tmp/mihomo
chmod +x /tmp/mihomo
/tmp/mihomo -v
install -m 0755 /tmp/mihomo /usr/local/bin/mihomo
```

创建配置目录：

```bash
install -d -m 0755 /etc/mihomo
```

## 拉取订阅配置

不要把真实订阅链接直接写进文档或脚本，建议通过环境变量传入：

```bash
export SUB_URL='https://example.com/api/v1/client/subscribe?token=REDACTED'
```

有些订阅链接默认返回 base64 节点列表，不是 Mihomo 可直接读取的 YAML。可以使用 Clash/Mihomo 的 User-Agent 拉取：

```bash
curl -L -A 'clash.meta' "$SUB_URL" -o /etc/mihomo/config.yaml
```

验证配置：

```bash
/usr/local/bin/mihomo -t -d /etc/mihomo
```

首次运行可能会下载 GeoIP 或规则数据库到配置目录。

## 配置 systemd 服务

创建服务文件：

```bash
vi /etc/systemd/system/mihomo.service
```

写入：

```ini
[Unit]
Description=Mihomo Proxy Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
LimitNPROC=500
LimitNOFILE=1000000
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

启动并设置开机自启：

```bash
systemctl daemon-reload
systemctl enable --now mihomo
```

常用管理命令：

```bash
systemctl status mihomo
systemctl restart mihomo
systemctl stop mihomo
systemctl start mihomo
```

## 配置终端代理开关

Mihomo 默认提供本地混合代理端口 `7890`。可以在 `~/.bashrc` 添加两个 alias：

```bash
alias proxy='export http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890 HTTPS_PROXY=http://127.0.0.1:7890 ALL_PROXY=socks5://127.0.0.1:7890'
alias unproxy='unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY'
```

让当前 shell 立即生效：

```bash
source ~/.bashrc
```

使用方式：

```bash
proxy
curl -I https://www.google.com
unproxy
```

这里的 `proxy` / `unproxy` 只控制当前 shell 是否使用本地代理，不改变 Mihomo 内部的规则模式。

## 规则分流与全局模式

Mihomo 的控制 API 通常监听在：

```text
127.0.0.1:9090
```

查看当前模式：

```bash
curl --noproxy '*' -sS http://127.0.0.1:9090/configs | rg -o '"mode":"[^"]+"'
```

切换到规则分流：

```bash
curl --noproxy '*' -sS -X PATCH http://127.0.0.1:9090/configs \
  -H 'Content-Type: application/json' \
  -d '{"mode":"rule"}'
```

切换到全局模式：

```bash
curl --noproxy '*' -sS -X PATCH http://127.0.0.1:9090/configs \
  -H 'Content-Type: application/json' \
  -d '{"mode":"global"}'
```

全局模式下还需要确认 `GLOBAL` 代理组没有选到 `DIRECT`。假设主代理组名称是 `<PROXY_GROUP>`：

```bash
curl --noproxy '*' -sS -X PUT 'http://127.0.0.1:9090/proxies/GLOBAL' \
  -H 'Content-Type: application/json' \
  -d '{"name":"<PROXY_GROUP>"}'
```

## 代理组策略

常见订阅配置里会包含几类代理组：

- `select`：手动选择节点或策略组
- `url-test`：按延迟自动选择可用节点
- `fallback`：按顺序使用第一个可用节点，故障时切换到下一个

规则分流模式下，常见行为是：

- 本地域名和局域网地址走 `DIRECT`
- 国内 IP 或国内域名走 `DIRECT`
- 其他流量走指定代理组

具体规则以 `/etc/mihomo/config.yaml` 为准。

## 切换节点

查看所有代理组：

```bash
curl --noproxy '*' -sS http://127.0.0.1:9090/proxies
```

假设主代理组名称是 `<PROXY_GROUP>`，要切到自动选择策略：

```bash
curl --noproxy '*' -sS -X PUT 'http://127.0.0.1:9090/proxies/<URL_ENCODED_PROXY_GROUP>' \
  -H 'Content-Type: application/json' \
  -d '{"name":"<AUTO_SELECT_GROUP>"}'
```

切到某个具体节点：

```bash
curl --noproxy '*' -sS -X PUT 'http://127.0.0.1:9090/proxies/<URL_ENCODED_PROXY_GROUP>' \
  -H 'Content-Type: application/json' \
  -d '{"name":"<NODE_NAME>"}'
```

如果代理组名称里有空格、中文或特殊字符，URL 路径里的代理组名需要进行 URL 编码。

## 更新订阅

重新设置订阅链接：

```bash
export SUB_URL='https://example.com/api/v1/client/subscribe?token=REDACTED'
```

拉取新配置并验证：

```bash
curl -L -A 'clash.meta' "$SUB_URL" -o /etc/mihomo/config.yaml
/usr/local/bin/mihomo -t -d /etc/mihomo
```

重启服务：

```bash
systemctl restart mihomo
```

检查代理是否可用：

```bash
systemctl status mihomo
curl -x http://127.0.0.1:7890 -I https://www.google.com
```

## 常见问题

如果 `mihomo -t` 提示订阅格式无法解析，通常是订阅链接返回了 base64 节点列表。优先尝试使用 `-A 'clash.meta'` 重新拉取。

如果控制 API 访问失败，检查配置文件里是否包含类似配置：

```yaml
external-controller: '127.0.0.1:9090'
```

如果全局模式仍然直连，检查 `GLOBAL` 代理组当前是否选中了 `DIRECT`。全局模式只决定匹配方式，最终出口仍由 `GLOBAL` 组的选择决定。

## 安全建议

- 订阅链接中的 `token` 等同于账户凭证，不要公开。
- `/etc/mihomo/config.yaml` 可能包含节点密码、UUID、服务地址等敏感信息，不要提交到 git。
- 如果订阅链接泄露，应在服务商后台重置订阅。
- 公开博客中不要出现真实订阅域名、真实代理组名、真实节点名或任何认证字段。
