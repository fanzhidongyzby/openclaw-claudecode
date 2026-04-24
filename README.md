# openclaw-claudecode

通过 tmux 操作 Claude Code 执行开发任务的技能。

## 链接

- **ClawHub**: https://clawhub.ai/fanzhidongyzby/openclaw-claudecode
- **GitHub**: https://github.com/fanzhidongyzby/openclaw-claudecode

## 简介

本技能用于在容器环境中通过 tmux 会话操控 Claude Code (cc) 执行长时间开发任务，包括：

- 启动 CC 开发会话
- 发送开发指令
- 监控进度
- 自动确认提示

## 快速开始

### 环境检查

首次使用必须检查环境：

```bash
# 一键检查脚本
check_env() {
  echo "=== tmux ==="
  which tmux && tmux -V || echo "❌ tmux missing"

  echo "=== claude ==="
  which claude && claude --version || echo "❌ claude missing"

  echo "=== user ==="
  id claudecode || echo "❌ user missing"

  echo "=== auth ==="
  claude --print "test" 2>&1 | grep -E "error|Error" && echo "❌ auth failed" || echo "✅ auth ok"
}
check_env
```

### 安装依赖

```bash
# 安装 tmux
dnf install -y tmux  # CentOS/RHEL/Rocky
apt install -y tmux  # Ubuntu/Debian

# 安装 Claude Code
npm install -g @anthropic-ai/claude-code

# 创建普通用户（CC 禁止 root 运行）
useradd -m -s /bin/bash claudecode
```

### 从 ClawHub 安装

```bash
# 安装技能
claw install fanzhidongyzby/openclaw-claudecode
```

### 启动开发任务

```bash
# 1. 创建 tmux 会话
tmux new-session -d -s cc-myproject -c /path/to/project

# 2. 启动 CC（普通用户）
tmux send-keys -t cc-myproject "su - claudecode -c 'cd /path/to/project && claude --dangerously-skip-permissions'" Enter

# 3. 等待 CC 启动
sleep 5

# 4. 发送开发任务
tmux send-keys -t cc-myproject "完成项目单元测试" Enter

# 5. 监控进度
tmux capture-pane -t cc-myproject -p | tail -20
```

## 核心命令

| 命令 | 用途 |
|------|------|
| `tmux new-session -d -s NAME -c DIR` | 创建后台会话 |
| `tmux send-keys -t NAME "CMD" Enter` | 发送命令 |
| `tmux capture-pane -t NAME -p` | 捕获输出 |
| `tmux list-sessions` | 列出会话 |
| `tmux attach -t NAME` | 附加会话 |
| `tmux kill-session -t NAME` | 杀死会话 |
| `tmux send-keys -t NAME y Enter` | 发送确认 |

## 文件说明

| 文件 | 说明 |
|------|------|
| `SKILL.md` | 中文版技能文档 |
| `SKILL_EN.md` | 英文版技能文档 |

## 常见问题

### CC 拒绝以 root 运行

切换到普通用户：

```bash
tmux send-keys -t <session> "su - claudecode -c 'cd /project && claude --dangerously-skip-permissions'" Enter
```

### 504 Gateway Timeout

等待 1-2 分钟后重启 CC：

```bash
tmux send-keys -t <session> C-c
tmux send-keys -t <session> "claude --dangerously-skip-permissions" Enter
```

### CC 卡在确认提示

自动确认：

```bash
tmux send-keys -t <session> y Enter
```

## 最佳实践

1. 首次使用必须检查环境：tmux、claude、普通用户、认证
2. 会话命名清晰：`cc-<项目>-<任务>`
3. 定期检查进度：每 30 分钟用 `capture-pane` 查看输出
4. 记录会话 ID：在 memory 文件中记录会话名和启动时间

## 作者

FlorianFan

## 许可证

MIT