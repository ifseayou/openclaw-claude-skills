---
name: claude-code
description: 通过 Claude Code CLI 执行编程任务：代码生成、Bug修复、重构、测试、PR创建等。支持一键调用和后台长任务。融合 Spec-Driven Development 工作流。
metadata:
  {
    "openclaw": {
      "emoji": "💻",
      "requires": { "anyBins": ["claude"] }
    },
  }
---

# Claude Code Skill

通过 Claude Code CLI（`claude`）在指定项目目录中执行编程任务。支持两种工作模式：**直接编码**和 **Spec-Driven Development（SDD）**。

## 前置条件

- Claude Code CLI 已安装且已登录（`claude auth status` 检查）
- 目标项目必须是 git 仓库
- SDD 模式需要安装 `specify-cli`（`uv tool install specify-cli --from git+https://github.com/github/spec-kit.git`）

---

## 第一部分：直接编码模式

适合快速修复、小功能、一次性任务。

### 1. 快速一次性任务（Print 模式）

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

```bash
exec pty:true workdir:<项目路径> background:true command:"claude -p '你的复杂任务描述' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions --output-format text"
```

监控和管理：
```
process action:poll sessionId:<id>     # 检查是否完成
process action:log sessionId:<id>      # 查看输出
process action:kill sessionId:<id>     # 终止任务
```

### 3. 常用任务模板

**Bug 修复：**
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '修复这个 bug：<描述>。先理解代码，找到根因，然后修复并验证。' --allowedTools 'Bash,Read,Edit,Write'"
```

**代码审查：**
```bash
exec pty:true workdir:<项目路径> timeout:120 command:"claude -p '审查最近的改动（git diff HEAD~1），关注质量、安全性和性能。' --allowedTools 'Bash,Read' --output-format json"
```

**写测试：**
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '为 <文件> 编写单元测试，覆盖主要逻辑和边界情况。' --allowedTools 'Bash,Read,Edit,Write'"
```

**创建 PR：**
```bash
exec pty:true workdir:<项目路径> timeout:120 command:"claude -p '将当前改动创建 PR：commit、push、gh pr create。' --allowedTools 'Bash(git *),Bash(gh *)' --dangerously-skip-permissions"
```

---

## 第二部分：Spec-Driven Development（SDD）模式

适合中大型功能开发。将「规格说明」作为核心驱动力，代码是规格的实现产物。

### SDD 核心理念

传统开发：想法 → 写代码 → 希望没 bug
SDD 开发：想法 → 规格说明 → 技术方案 → 任务拆分 → 自动实现

**规格说明（Spec）是真正的源代码，代码只是它的表达形式。**

### SDD 完整工作流（6 步）

#### 步骤 0：初始化 SDD 项目

对新项目或已有项目启用 SDD 支持：

```bash
# 新项目
exec pty:true timeout:60 command:"uvx --from git+https://github.com/github/spec-kit.git specify init <项目名> --ai claude"

# 已有项目
exec pty:true workdir:<项目路径> timeout:60 command:"uvx --from git+https://github.com/github/spec-kit.git specify init --here --ai claude --force"
```

这会在项目中创建 `.specify/` 目录结构和 Claude Code slash commands。

#### 步骤 1：建立项目宪法（Constitution）

定义项目的治理原则和开发准则：

```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.constitution 建立项目原则：代码质量优先、完善的测试覆盖、用户体验一致性、性能要求' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

输出：`.specify/memory/constitution.md`

#### 步骤 2：创建功能规格（Specify）

用自然语言描述你要构建的功能（关注「是什么」和「为什么」，不涉及技术栈）：

```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.specify <功能描述>' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

自动完成：
- 创建功能分支（如 `001-user-auth`）
- 生成 `specs/<分支名>/spec.md`（用户故事、验收标准、成功指标）
- 生成质量检查清单

#### 步骤 3：制定技术方案（Plan）

提供技术栈和架构选择：

```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.plan 使用 React + TypeScript 前端，FastAPI 后端，PostgreSQL 数据库' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

自动生成：
- `specs/<分支名>/plan.md`（技术方案）
- `specs/<分支名>/research.md`（技术调研）
- `specs/<分支名>/data-model.md`（数据模型）
- `specs/<分支名>/contracts/`（API 契约）
- `specs/<分支名>/quickstart.md`（验证场景）

#### 步骤 4：拆分任务（Tasks）

从技术方案自动生成可执行的任务清单：

```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.tasks' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

输出 `specs/<分支名>/tasks.md`：
- 按用户故事分阶段（Setup → Foundation → 各 User Story → Polish）
- 标记可并行任务 `[P]`
- 每个任务都有明确的文件路径和验收标准

#### 步骤 5：自动实现（Implement）

按照任务清单自动编码（这一步建议后台运行）：

```bash
exec pty:true workdir:<项目路径> background:true timeout:1800 command:"claude -p '/speckit.implement' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

自动完成：
- 检查清单合规性
- 按阶段执行每个任务
- 实时标记已完成任务 `[X]`
- 验证实现与规格的一致性

### SDD 可选步骤

**澄清需求（在 Plan 之前）：**
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.clarify' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

**一致性分析（在 Implement 之前）：**
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.analyze' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

**质量检查清单：**
```bash
exec pty:true workdir:<项目路径> timeout:300 command:"claude -p '/speckit.checklist <领域>' --allowedTools 'Bash,Read,Edit,Write' --dangerously-skip-permissions"
```

---

## 何时用哪种模式？

| 场景 | 模式 | 理由 |
|------|------|------|
| 修 bug、小调整 | 直接编码 | 快速高效 |
| 加个小功能（< 2 个文件） | 直接编码 | 不需要规划 |
| 代码审查 / PR | 直接编码 | 一次性任务 |
| 新功能（多文件、多模块） | SDD | 需要规划和任务拆分 |
| 新项目从零开始 | SDD | 完整的规格→实现流程 |
| 大规模重构 | SDD | 需要明确的方案和步骤 |
| 团队协作的功能 | SDD | 规格可版本化、可审查 |

---

## ⚠️ 重要规则

1. **必须使用 `pty:true`**：Claude Code 是交互式终端应用，没有 PTY 会挂起
2. **必须指定 `workdir`**：确保在正确的项目目录中工作
3. **一次性任务用 `-p`**：Print 模式自动退出，不会卡住
4. **长任务用 `background:true`**：避免阻塞，可通过 process 工具监控
5. **`--dangerously-skip-permissions`**：仅在信任的项目中使用
6. **不要在 OpenClaw 自身目录运行**：避免干扰运行
7. **rm/rm -rf 要谨慎**：优先使用 `trash` 代替

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

## 结合 OpenClaw 使用

当 Andy 请求编程任务时：
1. 确认目标项目路径（默认 `/Users/xhl/vibe_code`）
2. 判断复杂度：简单 → 直接编码，复杂 → SDD 工作流
3. 执行 Claude Code 并返回结果
4. 后台任务完成后主动汇报

### 自动通知完成

后台长任务可追加 wake 触发：
```
... 你的任务描述。

完成后运行: openclaw gateway wake --text "Done: [简要总结]" --mode now
```
