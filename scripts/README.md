脚本与剧本（Ansible / Bash / Python）。使用前先读懂逻辑，加固类操作先测一台。

## P2 基线加固剧本

文件：

```text
p2-hardening.yml
```

覆盖内容：

```text
系统安全更新
libsqlite3-0 修复
SSH 弱 MAC 禁用
SSH 基础加固
TCP timestamps 关闭
UFW 启用的安全顺序
ICMP timestamp 拦截
fail2ban
密码策略
auditd / sysstat
core dump 限制
登录警告横幅
```

执行前要求：

1. 三台 target 已打快照 `before-p2-hardening`。
2. 先在 `p2-target-1` 上执行。
3. 语法检查通过后再实际运行。
4. 验证 SSH、Ansible 和 GVM 凭据仍可用后，再批量执行。

建议命令：

```bash
ansible-playbook -i inventory.ini p2-hardening.yml --syntax-check
ansible-playbook -i inventory.ini p2-hardening.yml --limit p2-target-1 -K --diff
```

验证通过后再运行：

```bash
ansible-playbook -i inventory.ini p2-hardening.yml -K --diff
```
