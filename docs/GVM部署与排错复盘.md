# P2 GVM 部署与排错复盘

## 一、为什么这一段容易让人迷糊

GVM 部署和前面 Ansible、Lynis 的学习感受不同：

- Ansible 的目标清晰：批量管理服务器。
- Lynis 的目标清晰：检查主机安全基线。
- GVM 部署首先要解决很多环境问题，真正的漏洞扫描反而还没开始。

因此这一段主要是在学习：

```text
容器化部署
服务依赖关系
网络与 DNS
代理和 Fake-IP
磁盘与 LVM 扩容
SSH 隧道
服务健康检查
漏洞库导入
```

这些属于安全平台运维和排错能力，不是漏洞验证本身。

## 二、GVM 是做什么的

GVM 是 Greenbone Vulnerability Management，用于在授权范围内发现目标主机、服务和软件中的安全漏洞。

它与 Lynis 的区别：

| 工具 | 主要检查对象 | 输出 |
|---|---|---|
| Lynis | 主机配置和安全基线 | hardening index、warning、suggestion |
| GVM | 网络服务、软件版本、CVE、补丁和配置 | 漏洞、CVE、严重等级、修复建议 |

两者互补，不能互相替代。

## 三、GVM 主要组件

| 组件 | 作用 |
|---|---|
| `pg-gvm` | 保存用户、任务、配置和扫描结果 |
| `redis-server` | 保存扫描过程中的临时数据 |
| `gvmd` | GVM 管理核心和任务调度器 |
| `ospd-openvas` / `openvasd` | 执行漏洞检测的扫描器 |
| `gsad` / `gsa` / nginx | Web 控制台和 API |
| NVT Feed | 漏洞检测规则和测试脚本 |
| SCAP Feed | CVE、CPE 和漏洞关联数据 |
| CERT / DFN-CERT Feed | 厂商安全公告 |
| Notus Feed | 软件包级漏洞检测数据 |
| GVM Data | 扫描配置、端口列表和报告格式 |

可以简化理解为：

```text
Web 控制台 = 操作入口
gvmd = 总调度
OpenVAS = 实际扫描器
PostgreSQL = 数据库
漏洞 Feed = 扫描器的知识库
```

## 四、本次部署和排错时间线

### 1. 安装 Docker

使用 Ubuntu 软件源安装：

```bash
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
```

验证：

```bash
docker --version
sudo docker compose version
sudo docker run --rm hello-world
```

结论：Docker、Compose 和容器运行能力正常。

### 2. 下载 Greenbone Compose 文件

```bash
export DOWNLOAD_DIR="$HOME/greenbone-community-edition"
mkdir -p "$DOWNLOAD_DIR"
curl -f -L -o "$DOWNLOAD_DIR/compose.yaml" https://greenbone.github.io/docs/latest/_static/compose.yaml
```

Compose 文件负责编排 GVM 的全部容器和依赖关系。

### 3. 镜像下载失败

故障现象：

```text
failed to copy ... connection reset by peer
```

域名解析结果：

```text
registry.community.greenbone.net -> 198.18.1.128
```

原因：主机代理软件使用 Fake-IP，虚拟机缓存了该地址；关闭 TUN 后代理路由消失，Docker 仍访问旧的 Fake-IP。

处理：

- 暂时关闭 TUN。
- 清理虚拟机 DNS 缓存。
- 确认域名解析为真实地址。
- 使用低并发重新拉取。

```bash
sudo resolvectl flush-caches
getent ahostsv4 registry.community.greenbone.net
sudo env COMPOSE_PARALLEL_LIMIT=1 docker compose -f "$DOWNLOAD_DIR/compose.yaml" pull
```

### 4. GVM 容器启动

```bash
sudo docker compose -f "$DOWNLOAD_DIR/compose.yaml" up -d
sudo docker compose -f "$DOWNLOAD_DIR/compose.yaml" ps
```

一次 `scap-data` 健康检查在启动窗口内失败，导致 Compose 报告依赖错误；健康检查完成后重新执行 `up -d`，全部服务正常。

### 5. 根分区空间不足

导入数据时，根分区从约 29G 增长到 40G，只剩余约 6.3G。

由于虚拟机已有快照，没有直接修改原系统盘，而是新增一块 60GB 虚拟磁盘：

```bash
sudo pvcreate /dev/sdb
sudo vgextend ubuntu-vg /dev/sdb
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h /
```

扩容后根分区约 107G，空间重新充足。

### 6. 访问 GVM Web 控制台

GVM 的 nginx 只绑定在 mgmt 的 `127.0.0.1:443`，不能直接从主机访问。使用 SSH 本地端口转发：

```bash
ssh -N -L 9444:127.0.0.1:443 lab@192.168.10.10
```

浏览器访问：

```text
https://127.0.0.1:9444
```

重启 mgmt 后，SSH 隧道会断开，需要重新执行该命令。

### 7. 漏洞库导入

`Feed Status` 需要等待：

```text
NVT       Current
SCAP      Current
CERT      Current
GVM_DATA  Current
```

`gvmd` 重建 SCAP 数据库时会持续占用 CPU 和磁盘 I/O，这是正常现象。

## 五、本次出现的报错及其影响

| 报错 | 原因 | 是否有影响 |
|---|---|---|
| `No such container: gvmd` | `docker stats` 需要容器名，不是 Compose 服务名 | 无影响，改用 `docker compose stats` |
| `No SCAP database found` | 首次启动时 SCAP 数据库尚未建立 | 正常，随后会重建 |
| `Invalid characters in user name` | 初始化阶段用户创建逻辑报错，但 `admin` 已存在 | 无影响，已验证可登录 |
| `connection reset by peer` | 代理 Fake-IP 或节点连接被重置 | 会影响镜像下载，需要修复 |
| `connection refused` | DNS 还指向代理 Fake-IP，但 TUN 已关闭 | 会影响下载，需要清缓存 |
| `scap-data unhealthy` | 启动时健康检查尚未通过 | 通常是暂时的，重试 `up -d` |

## 六、这一段真正应该掌握什么

不需要背下所有容器名和 Compose 参数，应该掌握以下模型：

```text
目标主机
→ 扫描器
→ 漏洞库
→ 数据库
→ Web 控制台
→ 报告
```

遇到问题时先判断处于哪一层：

```text
网络层：DNS、NAT、代理、路由
系统层：CPU、内存、磁盘
容器层：镜像、容器、健康检查
服务层：数据库、gvmd、扫描器
应用层：Web UI、登录、任务
业务层：Target、Credential、Scan Task、Report
```

## 七、真正开始漏洞管理的下一步

漏洞库就绪后：

1. 检查 SSH Credential。
2. 检查 Target 是否包含三台主机。
3. 创建 Scan Task。
4. 选择目标、扫描器、扫描配置和端口范围。
5. 执行扫描并等待完成。
6. 分析漏洞、严重等级和修复建议。
7. 验证结果是否误报。
8. 制定加固计划。
9. 使用 Ansible 加固。
10. 复测并对比结果。

## 八、自测问题

1. Lynis 和 GVM 的检查范围有什么不同？
2. GVM 中 `gvmd`、OpenVAS、PostgreSQL 分别做什么？
3. 为什么漏洞扫描前需要建立 SSH Credential？
4. `Feed Status` 没有全部为 `Current` 时为什么不适合扫描？
5. 为什么 GVM 的 Web 控制台要通过 SSH 隧道访问？
6. 根分区不足时，为什么选择增加虚拟磁盘而不是直接修改带快照的原磁盘？
7. 扫描完成后，为什么不能直接把所有漏洞都判定为真实问题？
8. 加固后为什么要再次扫描，而不是只运行一次？

## 九、当前结论

Greenbone 部署阶段主要训练的是：

```text
平台部署
组件理解
网络排错
资源管理
安全访问
漏洞库管理
```

真正的漏洞发现、验证、修复和复测，将在 Feed 就绪并创建扫描任务后开始。
