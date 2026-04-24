---
name: openclaw-claudecode
description: |
  通过 tmux 操作 Claude Code (cc) 执行开发任务。适用于：启动 CC 开发会话、发送开发指令、监控进度、自动确认提示。
  触发场景：用户要求"启动 CC 开发"、"用 Claude Code 开发"、"tmux + cc 执行任务"、"监控 CC 进度"、"检查 CC 输出"。
  不适用于：简单代码编辑（直接用 edit 工具）、非开发类 Claude 对话。
metadata:
  author: FlorianFan
  version: "1.1.0"
---

# openclaw-claudecode Skill

通过 tmux 会话操控 Claude Code 执行长时间开发任务。

## 环境检查与初始化（首次使用必须执行）

### 0.1 检查 tmux

```bash
# 检查 tmux 是否安装
which tmux || echo "tmux not found"

# 若未安装，安装 tmux
# CentOS/RHEL/Rocky:
dnf install -y tmux || yum install -y tmux

# Ubuntu/Debian:
apt install -y tmux

# Alpine:
apk add tmux
```

### 0.2 检查 Claude Code

```bash
# 检查 claude 命令是否可用
which claude || echo "claude not found"

# 若未安装，安装 Claude Code
npm install -g @anthropic-ai/claude-code

# 或使用 claude 官方安装脚本
curl -fsSL https://claude.ai/install.sh | sh
```

### 0.3 验证 Claude Code 可用

```bash
# 检查版本
claude --version

# 测试简单调用（需要有效认证）
claude --print "hello" 2>&1 | head -5
```

**常见问题**：
- `Not logged in` → 需要先执行 `claude /login` 或配置 `ANTHROPIC_API_KEY`
- `model not found` → 检查 API 端点是否支持指定模型

### 0.4 创建普通用户（CC 禁止 root 运行）

```bash
# 检查是否存在普通用户
id <普通用户名> || echo "user not found"

# 创建用户（若不存在）
useradd -m -s /bin/bash <普通用户名>

# 设置密码（可选）
echo "<普通用户名>:<密码>" | chpasswd
```

**推荐用户名**：`claudecode`、`developer`、`ccdev`

### 0.5 环境检查汇总

执行完整环境检查：

```bash
# 一键检查脚本
check_env() {
  echo "=== tmux ==="
  which tmux && tmux -V || echo "❌ tmux missing"

  echo "=== claude ==="
  which claude && claude --version || echo "❌ claude missing"

  echo "=== user ==="
  id <普通用户名> || echo "❌ user missing"

  echo "=== auth ==="
  claude --print "test" 2>&1 | grep -E "error|Error" && echo "❌ auth failed" || echo "✅ auth ok"
}
check_env
```

**所有检查通过后才能执行后续流程**

---

## 核心流程

### 1. 定位项目目录

```bash
# 检查目录存在
ls -la <项目路径>

# 检查是否为 git 仓库（CC 需要）
ls -la <项目路径>/.git
```

**找不到目录 → 立即反馈用户确认路径**

### 2. 创建 tmux 会话

```bash
# 创建会话（指定工作目录）
tmux new-session -d -s <会话名> -c <项目路径>

# 验证会话
tmux list-sessions
```

**会话命名建议**：`cc-<项目名>` 或 `claude-<任务名>`

### 3. 切换用户 + 启动 CC

CC 禁止以 root 运行，必须切换到普通用户：

```bash
# 发送命令：切换用户 + cd + 启动 CC
tmux send-keys -t <会话名> "su - <普通用户> -c 'cd <项目路径> && claude --dangerously-skip-permissions'" Enter
```

**关键参数**：
- `--dangerously-skip-permissions`：跳过所有确认提示，全自动执行
- `--print`：静默模式，缓冲输出直到完成（适合后台任务）

### 4. 发送开发指令

```bash
# 单行指令
tmux send-keys -t <会话名> "<开发提示词>" Enter

# 多行指令（使用 load-buffer）
cat << 'EOF' | tmux load-buffer -
第一行指令
第二行指令
第三行指令
EOF
tmux paste-buffer -t <会话名>
tmux send-keys -t <会话名> Enter
```

### 5. 监控进度

```bash
# 查看最近输出
tmux capture-pane -t <会话名> -p | tail -30

# 查看完整滚动历史
tmux capture-pane -t <会话名> -p -S -

# 检查是否需要输入
tmux capture-pane -t <会话名> -p | tail -10 | grep -E "❯|Yes.*No|proceed|permission|y/n"
```

### 6. 自动确认/交互

```bash
# 发送确认
tmux send-keys -t <会话名> y Enter

# 发送取消
tmux send-keys -t <会话名> n Enter

# 发送 Ctrl+C 中断
tmux send-keys -t <会话名> C-c
```

---

## 完整示例

### 启动开发任务（含环境检查）

```bash
# 0. 环境检查
which tmux || dnf install -y tmux
which claude || npm install -g @anthropic-ai/claude-code
id claudecode || useradd -m claudecode

# 1. 创建会话
tmux new-session -d -s cc-<项目名> -c <项目路径>

# 2. 启动 CC（普通用户）
tmux send-keys -t cc-<项目名> "su - claudecode -c 'cd <项目路径> && claude --dangerously-skip-permissions'" Enter

# 3. 等待 CC 启动（约 5 秒）
sleep 5

# 4. 发送开发任务
tmux send-keys -t cc-<项目名> "<开发提示词>" Enter

# 5. 监控进度
tmux capture-pane -t cc-<项目名> -p | tail -20
```

### 检查 CC 进程状态

```bash
# 检查 CC 是否在运行
ps aux | grep -i claude | grep -v grep

# 检查运行时长
ps -o pid,etime,cmd -p $(pgrep -f "claude --dangerously")
```

### 会话管理

```bash
# 列出所有会话
tmux list-sessions

# 列出所有 attach 客户端
tmux list-clients

# 附加到会话（交互式查看）
tmux attach -t <会话名>

# 杀死会话
tmux kill-session -t <会话名>

# 杀死其他 attach 客户端（保留自己的）
tmux detach-client -t <会话名> -a

# 从附加会话中脱离
# 快捷键: Ctrl+B 然后 D
```

---

## 常见问题

### CC 拒绝以 root 运行

**解决**：切换到非 root 用户

```bash
tmux send-keys -t <会话名> "su - <普通用户> -c 'cd <项目路径> && claude --dangerously-skip-permissions'" Enter
```

### 504 Gateway Timeout

**原因**：Claude API 服务端临时错误

**解决**：等待 1-2 分钟后重启 CC

```bash
# 中断当前进程
tmux send-keys -t <会话名> C-c

# 重新启动
tmux send-keys -t <会话名> "claude --dangerously-skip-permissions" Enter
```

### CC 卡在确认提示

**检测**：

```bash
tmux capture-pane -t <会话名> -p | tail -5 | grep -E "y/n|Yes|No|proceed"
```

**解决**：

```bash
tmux send-keys -t <会话名> y Enter
```

### attach 进程残留

**检测**：

```bash
tmux list-clients
ps aux | grep "tmux attach" | grep -v grep
```

**清理**：

```bash
# 杀死多余的 attach 进程
kill <PID1> <PID2>

# 或踢掉其他客户端
tmux detach-client -t <会话名> -a
```

### 查看 CC 完成状态

```bash
# 检查是否回到 shell prompt
tmux capture-pane -t <会话名> -p | tail -3 | grep -E "\\$|#|❯"

# 检查 git 提交
cd <项目路径> && git log --oneline -3
```

---

## 最佳实践

1. **首次使用必须检查环境**：tmux、claude、普通用户、认证
2. **会话命名清晰**：`cc-<项目>-<任务>` 便于管理多个开发任务
3. **定期检查进度**：每 30 分钟用 `capture-pane` 查看输出
4. **记录会话 ID**：在 memory 文件中记录会话名和启动时间
5. **清理残留 attach**：定期检查 `tmux list-clients` 并清理

---

## 参数速查表

| 命令 | 用途 |
|------|------|
| `tmux new-session -d -s NAME -c DIR` | 创建后台会话 |
| `tmux send-keys -t NAME "CMD" Enter` | 发送命令 |
| `tmux capture-pane -t NAME -p` | 捕获输出 |
| `tmux capture-pane -t NAME -p -S -` | 捕获完整历史 |
| `tmux list-sessions` | 列出会话 |
| `tmux list-clients` | 列出 attach 客户端 |
| `tmux kill-session -t NAME` | 杀死会话 |
| `tmux attach -t NAME` | 附加会话 |
| `tmux detach-client -t NAME -a` | 踢掉其他客户端 |
| `tmux send-keys -t NAME C-c` | 发送 Ctrl+C |
| `tmux send-keys -t NAME y Enter` | 发送确认 |
| `tmux send-keys -t NAME n Enter` | 发送取消 |

| 检查命令 | 用途 |
|----------|------|
| `which tmux` | 检查 tmux |
| `which claude` | 检查 claude |
| `claude --version` | 检查版本 |
| `id <用户>` | 检查用户存在 |
| `ps aux | grep claude` | 检查进程 |
