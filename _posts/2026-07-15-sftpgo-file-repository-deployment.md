---
layout: post
title: "使用 Docker Compose 部署 SFTPGo 文件仓库"
date: 2026-07-15 21:40:00 +0800
categories: [教程]
tags: [sftpgo, docker, sftp, file-server, backup]
---

> 文中的 `192.0.2.0/24` 为文档示例网段，部署时请替换为实际服务器地址。

## 1. 文档说明

本文档用于在 `192.0.2.20` 上通过 Docker Compose 部署和维护 SFTPGo 文件仓库。

示例部署基线：

| 项目 | 配置 |
| --- | --- |
| 服务器 | `192.0.2.20` |
| CPU 架构 | ARM64（aarch64） |
| 部署方式 | Docker Compose |
| SFTPGo 版本 | `2.7.1` Community Edition |
| Web 管理端口 | `19080` |
| SFTP 端口 | `12022` |
| Compose 目录 | `/opt/file-repository` |
| 配置及数据库 | `/opt/file-repository/config` |
| 文件数据 | `/opt/file-repository/data` |
| 容器名称 | `sftpgo` |

管理页面：

```text
http://192.0.2.20:19080/web/admin
```

用户文件页面：

```text
http://192.0.2.20:19080/web/client
```

## 2. 部署架构

```text
浏览器
  └─ HTTP 19080 ──> SFTPGo WebAdmin / WebClient

SFTP 客户端
  └─ TCP 12022 ──> SFTPGo SFTP 2022

192.0.2.20
  ├─ /opt/file-repository/docker-compose.yml
  ├─ /opt/file-repository/config
  │    ├─ sftpgo.db
  │    └─ SSH Host Keys
  └─ /opt/file-repository/data
       ├─ data/<用户名>
       ├─ shared
       └─ backups
```

容器删除或重新创建后，只要 `/opt/file-repository` 完整保留，用户、权限、配置和文件数据仍然存在。

## 3. 部署前检查

### 3.1 检查系统环境

```bash
uname -m
docker version
docker compose version
df -h /opt
```

确保：

- CPU 架构为 `aarch64` 或 `arm64`。
- Docker 和 Docker Compose 可以正常使用。
- `/opt` 有足够磁盘空间。
- `19080` 和 `12022` 未被其他服务占用。

```bash
ss -lntp | grep -E ':19080|:12022'
```

### 3.2 启用 Docker 开机自启

```bash
systemctl enable --now docker
systemctl is-enabled docker
systemctl is-active docker
```

### 3.3 创建持久化目录

SFTPGo 容器使用 UID/GID `1000:1000` 运行：

```bash
install -d -o 1000 -g 1000 -m 0750 /opt/file-repository/config
install -d -o 1000 -g 1000 -m 0750 /opt/file-repository/data
```

在宿主机上，目录所有者会显示为 UID `1000` 对应的本地用户名，这是正常的 UID 映射结果。不要对数据目录执行 `chmod -R 777`。

## 4. Docker Compose 配置

创建 `/opt/file-repository/docker-compose.yml`：

```yaml
services:
  sftpgo:
    image: drakkan/sftpgo:v2.7.1
    container_name: sftpgo
    restart: always
    ports:
      - "192.0.2.20:19080:8080"
      - "192.0.2.20:12022:2022"
    volumes:
      - /opt/file-repository/config:/var/lib/sftpgo
      - /opt/file-repository/data:/srv/sftpgo
    security_opt:
      - no-new-privileges:true
    logging:
      driver: json-file
      options:
        max-size: "20m"
        max-file: "5"
```

配置说明：

- 固定使用 `2.7.1`，避免普通重启导致意外升级。
- Web 和 SFTP 端口只绑定到服务器业务 IP。
- `restart: always` 用于服务器重启后自动恢复。
- `no-new-privileges` 阻止容器进程通过 setuid 等方式获得额外权限。
- Docker 日志最多保留 5 个文件，每个文件最大 20 MB。

## 5. 启动与验证

### 5.1 启动服务

```bash
cd /opt/file-repository
docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

### 5.2 检查日志

```bash
docker compose logs --tail 200 sftpgo
```

首次启动时，SFTPGo 会创建：

- SQLite 数据库 `sftpgo.db`。
- RSA、ECDSA 和 ED25519 SSH Host Key。
- 初始数据库表结构。

日志中首次出现 `no such table` 后紧接着创建数据库结构，属于首次初始化过程，并非持续故障。

### 5.3 验证运行状态

```bash
docker inspect sftpgo \
  --format 'status={{.State.Status}} restart={{.RestartCount}} policy={{.HostConfig.RestartPolicy.Name}}'

curl -I http://192.0.2.20:19080/web/admin
ss -lntp | grep -E ':19080|:12022'
```

正常情况下：

- 容器状态为 `running`。
- 重启策略为 `always`。
- `/web/admin` 跳转到管理员登录或首次初始化页面。
- `19080` 和 `12022` 均处于监听状态。

## 6. 首次初始化

首次访问：

```text
http://192.0.2.20:19080/web/admin/setup
```

按照页面提示创建第一个管理员账号。管理员密码应：

- 使用高强度独立密码。
- 不写入 Compose、脚本或代码仓库。
- 在条件允许时启用双因素认证。

管理员后台和用户页面不同：

| 页面 | 用途 |
| --- | --- |
| `/web/admin` | 管理用户、Group、Folder、权限和服务配置 |
| `/web/client` | 普通用户上传、下载、分享和管理文件 |

SFTPGo Community 当前没有官方中文界面。可使用浏览器的网页翻译功能，后端功能不受影响。

## 7. 创建文件用户

进入：

```text
WebAdmin → Users → +
```

至少填写：

- Username：用户登录名。
- Password 或 Public Keys：认证凭据。
- Home Dir：可留空使用默认规则，也可明确设置为 `/srv/sftpgo/data/<用户名>`。
- Permissions：根据需要授予上传、下载、删除等权限。
- Quota：根据用户用途设置容量和文件数量限制。

SFTP 登录示例：

```bash
sftp -P 12022 <用户名>@192.0.2.20
```

网页登录地址：

```text
http://192.0.2.20:19080/web/client
```

## 8. Group 与共享目录

### 8.1 Group 类型

| Group 类型 | 作用 |
| --- | --- |
| Primary | 每个用户最多一个，提供主目录、默认权限、配额等基础配置 |
| Secondary | 每个用户可以多个，用于追加共享目录、权限和过滤规则 |
| Membership | 只记录成员关系，不继承目录和权限 |

加入同一个 Group 不会自动共享用户各自的主目录。共享文件应使用 Virtual Folder，并通过 Primary 或 Secondary Group 分配给用户。

### 8.2 创建共享 Folder

进入：

```text
WebAdmin → Folders → +
```

示例配置：

| 字段 | 值 |
| --- | --- |
| Name | `team-shared` |
| Storage | Local filesystem |
| Absolute path | `/srv/sftpgo/shared/team` |
| Quota size | `0`，表示不限制 |
| Quota files | `0`，表示不限制 |

容器内路径：

```text
/srv/sftpgo/shared/team
```

对应主机路径：

```text
/opt/file-repository/data/shared/team
```

### 8.3 创建共享 Group

进入：

```text
WebAdmin → Groups → +
```

创建 `team` Group，在 Virtual Folders 中添加：

| 字段 | 值 |
| --- | --- |
| Folder | `team-shared` |
| Virtual path | `/shared` |
| Quota size/files | `0` |

在 ACL/Permissions 中为 `/shared` 设置权限：

- 完全读写：选择 `*`。
- 只读：只选择 `list` 和 `download`。
- 只允许上传：按实际需要选择 `list`、`upload`、`create_dirs` 等权限。

### 8.4 将用户加入 Group

进入：

```text
WebAdmin → Users → Edit → Secondary Groups
```

将 `team` 添加为 Secondary Group。用户重新登录后会看到：

```text
/
├── 用户自己的文件
└── shared
    └── Group 共享文件
```

所有成员访问的是同一个实际目录，一个成员上传的文件会立即对其他有权限的成员可见。

共享 Virtual Folder 不要设置成“计入单个用户配额”，否则多个用户共享时配额统计可能不准确。应为 Folder 使用独立配额。

如果需要读写组和只读组，可以创建 `team-rw` 与 `team-ro` 两个 Secondary Group，都映射同一个 Folder 到 `/shared`，但配置不同 ACL。不要让同一用户同时加入两个在相同 Virtual path 上定义不同映射的 Group。

## 9. 日常运维

```bash
cd /opt/file-repository

# 状态
docker compose ps

# 实时日志
docker compose logs -f --tail 200 sftpgo

# 重启
docker compose restart sftpgo

# 停止
docker compose stop sftpgo

# 启动
docker compose start sftpgo

# 按 Compose 配置重新创建
docker compose up -d sftpgo
```

查看资源使用：

```bash
docker stats --no-stream sftpgo
du -sh /opt/file-repository/config /opt/file-repository/data
df -h /opt
df -ih /opt
```

## 10. 开机自动恢复

检查 Docker 服务和容器重启策略：

```bash
systemctl is-enabled docker
systemctl is-active docker
docker inspect sftpgo --format '{{.HostConfig.RestartPolicy.Name}}'
```

预期结果为：

- Docker：`enabled`、`active`。
- 容器重启策略：`always`。

服务器重启后检查：

```bash
docker ps --filter name=sftpgo
curl -I http://192.0.2.20:19080/web/admin
```

## 11. 数据备份

### 11.1 备份内容

必须同时备份：

1. `/opt/file-repository/config`：数据库、SSH Host Key 和服务状态。
2. `/opt/file-repository/data`：所有用户文件和共享文件。
3. `/opt/file-repository/docker-compose.yml`：部署配置。

只备份文件目录而不备份数据库，会丢失用户、密码哈希、Group、Folder 和权限关系。

### 11.2 一致性备份

SQLite 数据库适合在短暂停止服务后进行文件级备份：

```bash
cd /opt/file-repository
docker compose stop sftpgo

tar -C /opt -czf /opt/sftpgo-backup-$(date +%F-%H%M%S).tar.gz \
  file-repository/config \
  file-repository/data \
  file-repository/docker-compose.yml

docker compose start sftpgo
```

确认服务恢复：

```bash
docker compose ps
curl -I http://192.0.2.20:19080/web/admin
```

### 11.3 备份原则

- 备份必须复制到另一台服务器、独立磁盘或对象存储。
- 仅放在 `/opt` 同一块磁盘上的备份不能防止磁盘损坏。
- 备份中包含密码哈希和 SSH Host Key，应限制访问权限并加密保存。
- 定期执行恢复演练。
- 备份前确认磁盘剩余空间。

## 12. 数据恢复

恢复会覆盖当前服务数据，应在维护窗口执行。

```bash
cd /opt/file-repository
docker compose stop sftpgo
```

将当前目录保留为恢复前快照，再从备份中恢复 `config`、`data` 和 Compose 文件。恢复后检查权限：

```bash
chown -R 1000:1000 /opt/file-repository/config
chown -R 1000:1000 /opt/file-repository/data
chmod 0750 /opt/file-repository/config
chmod 0750 /opt/file-repository/data
```

启动并验证：

```bash
cd /opt/file-repository
docker compose config
docker compose up -d
docker compose logs --tail 200 sftpgo
curl -I http://192.0.2.20:19080/web/admin
```

恢复后应验证：

- 管理员能够登录。
- 用户和 Group 配置存在。
- 用户自己的目录可以访问。
- Group 共享目录可以读写。
- SFTP Host Key 未意外改变。

## 13. 版本升级

升级原则：

- 不使用 `latest`。
- 升级前备份配置、数据库和文件数据。
- 阅读目标版本 Release Notes，特别关注数据库迁移和不兼容变更。
- 先在测试环境验证，再升级生产实例。

升级步骤：

```bash
cd /opt/file-repository

# 完成备份后，修改 docker-compose.yml 中的固定版本
docker compose config
docker compose pull sftpgo
docker compose up -d sftpgo

docker compose logs --tail 200 sftpgo
docker inspect sftpgo --format '{{.State.Status}} {{.RestartCount}}'
```

不要只依赖旧镜像回滚。若新版本已迁移数据库，应使用升级前完整备份恢复。

## 14. HTTPS 与安全加固

示例 Web 地址使用 HTTP：

```text
http://192.0.2.20:19080
```

HTTP 会明文传输登录凭据和文件内容，不应直接暴露到公网或不可信网络。

生产使用建议：

- 配置域名，例如 `files.example.internal`。
- 使用 Nginx、HAProxy 或负载均衡器提供 HTTPS。
- 使用客户端信任的内部 CA 或正式 CA 证书。
- 防火墙仅允许受信任网段访问 `19080` 和 `12022`。
- 管理员启用双因素认证。
- 普通用户优先使用 SSH 公钥认证。
- 为用户和共享 Folder 设置合理配额。
- 定期检查失败登录和封禁记录。
- 不向普通用户授予不必要的删除、覆盖和分享权限。
- 定期升级到包含安全修复的版本。

反向代理部署完成后，可只允许代理服务器访问 `19080`，避免用户绕过 HTTPS 直接访问 HTTP。

## 15. 常见问题

### 15.1 页面访问不了

```bash
docker compose -f /opt/file-repository/docker-compose.yml ps
docker logs --tail 200 sftpgo
ss -lntp | grep 19080
curl -I http://192.0.2.20:19080/web/admin
```

检查主机防火墙、客户端到服务器的路由和端口策略。

### 15.2 SFTP 无法连接

```bash
ss -lntp | grep 12022
ssh-keyscan -p 12022 192.0.2.20
sftp -vvv -P 12022 <用户名>@192.0.2.20
```

确认连接的是 `12022`，而不是 GitLab 使用的 `12222` 或主机 SSH 使用的 `22`。

### 15.3 用户看不到共享目录

检查：

- Folder 是否已创建。
- Group 是否引用该 Folder。
- Virtual path 是否设置为 `/shared` 等绝对路径。
- 用户是否加入了正确的 Primary 或 Secondary Group。
- 是否错误使用 Membership Group。
- 用户自身是否已在相同 Virtual path 配置了其他 Folder。
- 修改 Group 后用户是否重新登录。

### 15.4 共享目录只能查看不能上传

检查 Group 的 ACL/Permissions。只授予 `list` 和 `download` 时为只读，需要上传时还应授予 `upload`，根据使用场景增加 `overwrite`、`create_dirs`、`rename` 或 `delete`。

### 15.5 主机目录显示为本地用户名

容器以 UID/GID `1000:1000` 运行，宿主机执行 `ls` 时会显示 UID `1000` 对应的本地用户名。这是正常现象，不要随意改成 root 或开放为 `777`。

### 15.6 删除文件后磁盘空间没有明显下降

先检查是否仍有进程持有已删除文件，以及文件是否位于其他用户目录或备份中：

```bash
du -sh /opt/file-repository/data/*
lsof +L1
```

执行任何批量删除前，应先完成备份并确认路径范围。

## 16. 监控建议

至少监控：

- SFTPGo 容器是否运行、是否频繁重启。
- Web 页面和 SFTP 端口是否可用。
- `/opt` 磁盘和 inode 使用率。
- 用户与共享 Folder 配额。
- 登录失败、暴力破解和异常来源 IP。
- 最近一次备份时间、大小和异机复制状态。
- HTTPS 证书到期时间。

快速巡检：

```bash
docker inspect sftpgo \
  --format 'status={{.State.Status}} restart={{.RestartCount}} policy={{.HostConfig.RestartPolicy.Name}}'
docker stats --no-stream sftpgo
df -h /opt
df -ih /opt
du -sh /opt/file-repository/config /opt/file-repository/data
curl -I http://192.0.2.20:19080/web/admin
```

## 17. 官方参考资料

- SFTPGo 文档：https://docs.sftpgo.com/
- WebAdmin 与 WebClient：https://docs.sftpgo.com/latest/web-interfaces/
- Group：https://docs.sftpgo.com/Enterprise/groups/
- Virtual Folder：https://docs.sftpgo.com/enterprise/virtual-folders/
- Community 源码与版本：https://github.com/drakkan/sftpgo
- SFTPGo 2.7.1：https://github.com/drakkan/sftpgo/releases/tag/v2.7.1
