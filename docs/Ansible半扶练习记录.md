# Ansible 半扶练习记录：sysctl 配置下发

## 练习目标

通过一个小型 Playbook，理解：

```text
hosts
become
tasks
ansible.builtin.copy
ansible.builtin.command
模块参数
幂等性
--limit
--diff
-K
```

## 练习内容

新增两个内核安全参数：

```text
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
```

对应文件：

```text
scripts/practice-sysctl.yml
scripts/practice-sysctl.template.yml
```

## Playbook 结构

```text
Play
├── hosts: targets
├── become: true
└── tasks
    ├── copy
    │   ├── dest
    │   ├── owner
    │   ├── group
    │   ├── mode
    │   └── content
    └── command
        ├── sysctl --system
        └── changed_when: false
```

## 执行过程

### 1. 单台执行

```bash
ansible-playbook -i inventory.ini practice-sysctl.yml --limit p2-target-1 -K --diff
```

结果：

```text
copy 任务 changed
command 任务 ok
sysctl 参数验证成功
```

### 2. 幂等性验证

再次对 target1 执行：

```text
p2-target-1: changed=0
```

说明目标已经达到期望状态，没有重复修改文件。

### 3. 批量执行

```text
p2-target-1: changed=0
p2-target-2: changed=1
p2-target-3: changed=1
```

解释：

- target1 已经配置过，所以 `changed=0`。
- target2/3 第一次配置，所以 `changed=1`。
- 三台最终都达到期望状态。

## 验证结果

```text
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
```

三台 target 均通过验证。

## 学到的关键点

```text
YAML 使用空格缩进，不使用 Tab
模块参数必须缩进到模块下面
copy 模块负责声明目标文件内容
command 模块负责执行 sysctl --system
changed=0 表示目标已经是期望状态
--limit 用于缩小变更影响范围
-K 用于输入 sudo 密码
```
