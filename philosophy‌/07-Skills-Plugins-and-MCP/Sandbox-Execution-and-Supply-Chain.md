# Sandbox：执行隔离与供应链安全

## 本文档在系统中的作用

Sandbox 为不可信或高风险执行提供隔离边界，例如 Python 代码、第三方 Skill 脚本、文档转换、网页抓取和生成文件。目标不是“让代码能跑”，而是限制代码能看到什么、访问什么、消耗多少资源。

## 1. 主进程不执行不可信代码

禁止在 API / Agent Worker 主进程中对用户内容使用 `eval`、`exec`、动态 import 或 Shell。Python 语言层面的限制不足以构成安全沙箱。

## 2. Sandbox 规格

image_digest、command、read_only_rootfs、cpu_limit、memory_limit、pid_limit、wall_time_limit、filesystem_mounts、network_egress_policy、secret_mount_policy、output_limit、tenant_id、run_id。

## 3. 文件系统隔离

默认临时目录；只挂载本 Run 必需文件；不挂载宿主 Docker Socket；不挂载云凭证；不挂载其他 Tenant 目录；限制软链接和路径穿越；限制解压文件数量和总大小；输出目录单独挂载并扫描。

## 4. 网络隔离

默认无外网；确需访问时使用域名 / IP 白名单和受控代理。阻断私网、Loopback、云 Metadata Endpoint、宿主机和管理网段。对 DNS 解析结果进行校验，防止 SSRF 和 DNS Rebinding。

## 5. 任务状态

`created → queued → provisioning → running → collecting → scanning → succeeded / failed / timeout / killed`

Sandbox Worker 需要 Lease、Heartbeat 和 Reaper，处理 Worker 崩溃后的孤儿容器。

## 6. 供应链

镜像使用 Digest 固定；基础镜像定期扫描；依赖文件锁定；不允许运行时任意 pip install；若支持依赖安装，应使用预构建镜像或离线缓存，并执行许可证 / 漏洞扫描。

## 7. 输出处理

Sandbox 输出不是可信内容。文件需要病毒扫描、MIME 检查、路径检查和大小限制。输出中的命令或脚本内容不能自动执行。

## 8. 验收

尝试读取宿主文件、访问 Metadata Endpoint、访问其他 Tenant 文件、fork bomb、内存耗尽、无限循环、网络外传和路径穿越，确认隔离有效。
