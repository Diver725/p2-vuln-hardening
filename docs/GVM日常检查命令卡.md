# GVM 日常检查命令卡

> 用途：检查 Docker、GVM 容器、Web 入口和 Feed 状态。
> 不需要背完整命令，理解每个检查层次即可。

## 1. 检查 Docker 服务

```bash
systemctl is-active docker
```

正常输出：

```text
active
```

## 2. 检查 GVM 所有容器

```bash
export DOWNLOAD_DIR="$HOME/greenbone-community-edition"
sudo docker compose -f "$DOWNLOAD_DIR/compose.yaml" ps
```

重点容器：

```text
pg-gvm       healthy
gvmd         healthy
scap-data    healthy
gsa          healthy
nginx        Up
gsad         Up
ospd-openvas Up
openvasd     Up
```

## 3. 检查 GVM Web 入口

在 p2-mgmt 本机：

```bash
curl -kI https://127.0.0.1:443
```

预期看到：

```text
HTTP/1.1 200 OK
```

或：

```text
HTTP/1.1 302 Found
```

## 4. 查看关键日志

```bash
sudo docker compose -f "$DOWNLOAD_DIR/compose.yaml" logs --tail=100 gvmd
```

```bash
sudo docker compose -f "$DOWNLOAD_DIR/compose.yaml" logs --tail=100 ospd-openvas
```

```bash
sudo docker compose -f "$DOWNLOAD_DIR/compose.yaml" logs --tail=100 openvasd
```

如果日志持续更新，说明相关服务还在工作。

## 5. 检查 Feed Status

在 GVM 页面：

```text
Administration
→ Feed Status
```

正常：

```text
NVT       Current
SCAP      Current
CERT      Current
GVM_DATA  Current
```

以下状态表示暂时不适合扫描：

```text
Update in progress
Synchronization issue
Scanner still starting
3 days old
```

判断方式：

1. 先看 `docker compose ps` 是否健康。
2. 再看 `gvmd` / `ospd-openvas` / `openvasd` 日志是否在更新。
3. 如果服务健康且日志在更新，等待即可。
4. 如果长期不变且服务 unhealthy，再排查 Feed 或容器问题。

## 6. 主机访问隧道

主机 PowerShell：

```powershell
ssh -N -L 9777:127.0.0.1:443 lab@192.168.10.10
```

浏览器：

```text
https://127.0.0.1:9777
```

隧道窗口必须保持开启。

## 7. 最低限度需要记住的内容

不需要背完整命令，只需要记住：

```text
Docker 是否 active
Compose 容器是否 healthy
Web 443 是否可访问
Feed 是否 Current
日志是否在更新
```

具体命令可以直接查这张卡。
