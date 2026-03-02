---
name: claude-code
description: 通过 Claude Code CLI 执行编程任务：代码生成、Bug修复、重构、测试、PR创建等。支持一键调用和后台长任务。
metadata:
  {
    "openclaw": {
      "emoji": "💻",
      "requires": { "anyBins": ["claude"] }
    },
  }
---

# Claude Code Skill

通过 Claude Code CLI（`claude`）在指定项目目录中执行编程任务。

## 前置条件

- Claude Code CLI 已安装且已登录（`claude auth status` 检查）
- 目标项目必须是 git 仓库

## 核心用法

### 1. 快速一次性任务（Print 模式）

适合简单、快速的任务，直接返回结果：

```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '你的任务描述' --allowedTools 'Bash,Read,Edit,Write' --output-format json"
```

**关键参数：**
- `-p`（print 模式）：非交互式运行，完成后退出
- `--allowedTools`：自动授权工具，避免权限提示卡住
- `--output-format json`：返回结构化 JSON（含 `result`、`session_id`、`usage`）
- `--output-format text`：返回纯文本（默认）
- `--model`：指定模型（如 `opus`、`sonnet`、`claude-opus-4-6`）
- `--max-turns`：限制最大轮次（防止失控）
- `--max-budget-usd`：限制最大花费

### 2. 后台长任务

适合耗时较长的复杂任务（重构、大规模修改等）：

```bash
exec pty:true workdir:<项目路径> background:true command:"claude -p '你的复杂任务描述' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions --output-format text"
```

监控和管理后台任务：
```
process action:poll sessionId:<id>     # 检查是否完成
process action:log sessionId:<id>      # 查看输出
process action:kill sessionId:<id>     # 终止任务
```

### 3. 继续上一次对话

```bash
exec pty:true workdir:<项目路径> command:"claude -p '继续的内容' --continue --allowedTools 'Bash,Read,Edit,Write'"
```

### 4. 带系统提示的任务

```bash
exec pty:true workdir:<项目路径> command:"claude -p '任务' --append-system-prompt '使用 TypeScript，遵循项目现有风格' --allowedTools 'Bash,Read,Edit,Write'"
```

## 常用任务模板

### Bug 修复
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '修复这个 bug：<bug描述>。先理解代码，找到根因，然后修复并验证。' --allowedTools 'Bash,Read,Edit,Write'"
```

### 添加功能
```bash
exec pty:true workdir:<项目路径> timeout:600 background:true command:"claude -p '实现以下功能：<功能描述>。遵循项目现有架构和代码风格。' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

### 代码审查
```bash
exec pty:true workdir:<项目路径> timeout:120 command:"claude -p '审查最近的代码改动（git diff HEAD~1），关注代码质量、安全性和性能问题，给出改进建议。' --allowedTools 'Bash,Read' --output-format json"
```

### 写测试
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '为 <文件/模块> 编写单元测试，覆盖主要逻辑和边界情况，使用项目现有测试框架。' --allowedTools 'Bash,Read,Edit,Write'"
```

### 创建 PR
```bash
exec pty:true workdir:<项目路径> timeout:120 command:"claude -p '将当前改动创建一个 PR：生成合适的 commit message，push 到远程，然后用 gh 创建 PR。' --allowedTools 'Bash(git *),Bash(gh *)' --dangerously-skip-permissions"
```

### 代码重构
```bash
exec pty:true workdir:<项目路径> timeout:600 background:true command:"claude -p '重构 <模块>：<重构目标>。确保所有测试通过。' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

### 项目分析
```bash
exec pty:true workdir:<项目路径> timeout:120 command:"claude -p '分析这个项目的架构和代码结构，给出总结。' --allowedTools 'Bash,Read' --permission-mode plan --output-format text"
```

## ⚠️ 重要规则

1. **必须使用 `pty:true`**：Claude Code 是交互式终端应用，没有 PTY 会挂起
2. **必须指定 `workdir`**：确保 Claude Code 在正确的项目目录中工作
3. **一次性任务用 `-p`**：Print 模式自动退出，不会卡住
4. **长任务用 `background:true`**：避免阻塞，可通过 process 工具监控
5. **`--dangerously-skip-permissions`**：仅在信任的项目中使用，跳过所有权限提示
6. **`--allowedTools`**：更安全的替代方案，只授权特定工具
7. **不要在 OpenClaw 自身目录运行**：避免干扰 OpenClaw 运行

## 输出格式

### JSON 输出（`--output-format json`）
```json
{
  "type": "result",
  "result": "Claude 的回复文本",
  "session_id": "uuid",
  "usage": { "input_tokens": 1234, "output_tokens": 567 }
}
```

### Stream JSON（`--output-format stream-json`）
实时流式输出，每行一个 JSON 事件，适合需要实时反馈的场景。

## 结合 OpenClaw 使用

当 Andy 请求编程任务时：
1. 确认目标项目路径（默认 `/Users/xhl/vibe_code`）
2. 根据任务复杂度选择前台（简单）或后台（复杂）模式
3. 执行 Claude Code 并返回结果
4. 后台任务完成后主动汇报
