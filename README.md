# ask-codebuddy

**中文** | [English](#english)

> 把边界清晰的任务交给本机 CodeBuddy CLI，主代理负责拆分、核验与整合。
> Delegate well-scoped subtasks to your local CodeBuddy CLI — the main agent plans, verifies, and integrates.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 中文

### 这是什么

`ask-codebuddy` 是一个 Agent Skill（适用于 Codex / [CC] 等支持 `SKILL.md` 的客户端）。它把本机已登录的 CodeBuddy CLI 当作辅助子代理，用来处理可以独立完成的工作：代码分析、代码审查、方案比较、问题定位和局部实现，从而利用已有的 CodeBuddy 订阅或积分。

设计原则是**主代理负责，子代理执行**：

- 主代理理解需求、拆分任务、给出必要上下文和明确的可修改文件范围。
- CodeBuddy 只做被指派的那一部分，不继承对话上下文。
- 主代理检查证据、差异和测试结果，再决定是否采用。
- 简单任务直接完成，不为委派而委派。

### 前置条件

- 已安装 CodeBuddy CLI（命令 `codebuddy`，别名 `cbc`），常见路径为 `~/.local/bin/codebuddy`。
- 已在本机完成登录，且账户有可用额度。
- 能非交互调用，例如 `codebuddy --print --model custom-local:BC-DeepSeek-V4.1-Flash "hello"`。

### 安装

```bash
# Codex
git clone https://github.com/zcx960/ask-codebuddy.git ~/.codex/skills/ask-codebuddy

# [CC]
git clone https://github.com/zcx960/ask-codebuddy.git ~/.claude/skills/ask-codebuddy
```

也可以把仓库放到任意 skills 目录下，目录名保持 `ask-codebuddy` 即可。

### 使用

在对话中显式调用技能，并说明要委派的工作：

```text
$ask-codebuddy 审查 src/auth/ 下的登录逻辑，列出并发问题，不要改代码。
```

技能会按下面的流程工作：

1. **拆分**：判断任务是否适合委派，划出独立的文件或目录范围。
2. **传参**：把任务写进 `work/` 临时文件，用标准输入传给 CLI，避免拼接 shell 字符串。
3. **执行**：按需要选择工具集——纯分析用 `--tools ""`，只读审查用 `Read,Glob,Grep`，局部实现用 `Read,Glob,Grep,Edit,Write`。
4. **校验**：解析 JSON 输出，核对 `is_error`、`subtype`、`result` 和工具调用记录，再由主代理检查实际改动。

### 命令模板

纯分析（不读取文件、不执行命令）：

```bash
codebuddy \
  --print \
  --model custom-local:BC-DeepSeek-V4.1-Flash \
  --tools "" \
  --output-format json \
  --max-turns 4 \
  --setting-sources user \
  --strict-mcp-config \
  --mcp-config '{"mcpServers":{}}' \
  --no-session-persistence \
  < work/codebuddy-prompt.txt > work/codebuddy-result.json
```

只读审查：把 `--tools` 换成 `--tools "Read,Glob,Grep" --allowedTools "Read,Glob,Grep" --permission-mode dontAsk`。

局部实现：在独立目录或 Git worktree 中运行，使用 `--tools "Read,Glob,Grep,Edit,Write" --permission-mode acceptEdits`，并约定可修改的文件和完成条件。

### 参数说明

| 参数 | 作用 |
| --- | --- |
| `--print` | 非交互模式，输出结果后退出 |
| `--model custom-local:BC-DeepSeek-V4.1-Flash` | 显式指定模型，不改动全局默认配置 |
| `--tools ""` | 禁用全部工具，只处理提示里的内容 |
| `--output-format json` | 输出机器可解析的 JSON |
| `--max-turns 4` | 限制轮次，控制耗时和额度 |
| `--setting-sources user` | 只加载用户级设置 |
| `--strict-mcp-config` + 空 `--mcp-config` | 不加载任何 MCP 服务，避免误触外部系统 |
| `--no-session-persistence` | 不落盘会话，不污染历史记录 |

### 注意事项

- **不要只看退出码**。JSON 输出可能是事件数组，也可能是单个对象；要找到 `type: "result"` 的那条，检查 `is_error` 和 `subtype`。
- **不要用 `total_cost_usd` 推断积分消耗**，实际扣费以账户账单为准。
- **不继承上下文**。CodeBuddy 看不到当前对话，必须显式提供项目约束、文件路径和完成标准。
- **不要并发修改同一文件**。多个委派任务应划分互不冲突的范围。
- **委派不扩大授权**。发布、部署、远端变更和对外发送消息仍需单独授权。
- 遇到登录失效、额度不足或模型不可用时，如实说明错误，不自动重试到无限，也不擅自换模型或购买额度。

### 许可

[MIT](LICENSE) © 2026 zcx960

---

## English

### What this is

`ask-codebuddy` is an agent skill (works with Codex, [CC], and other clients that read `SKILL.md`). It uses your locally logged-in CodeBuddy CLI as a helper sub-agent for work that can stand on its own: code analysis, review, comparing approaches, bug hunting, and small scoped implementations — making use of an existing CodeBuddy subscription or credits.

The design principle is **the main agent owns the task, the sub-agent does the work**:

- The main agent understands the request, splits it up, and provides the context plus an explicit list of files that may change.
- CodeBuddy handles only that slice, and inherits none of the conversation context.
- The main agent checks the evidence, the diff, and the test results before accepting anything.
- Small tasks are done directly — no delegating for the sake of delegating.

### Requirements

- CodeBuddy CLI installed as `codebuddy` (alias `cbc`), commonly at `~/.local/bin/codebuddy`.
- Logged in on this machine, with available credits.
- Non-interactive calls work, e.g. `codebuddy --print --model custom-local:BC-DeepSeek-V4.1-Flash "hello"`.

### Install

```bash
# Codex
git clone https://github.com/zcx960/ask-codebuddy.git ~/.codex/skills/ask-codebuddy

# [CC]
git clone https://github.com/zcx960/ask-codebuddy.git ~/.claude/skills/ask-codebuddy
```

Any skills directory works, as long as the folder is named `ask-codebuddy`.

### Usage

Invoke the skill explicitly and describe the work to delegate:

```text
$ask-codebuddy Review the login logic under src/auth/ and list concurrency issues. Do not change code.
```

The skill then works in four steps:

1. **Split** — decide whether the task is worth delegating and carve out a self-contained file or directory scope.
2. **Hand off** — write the task to a temp file under `work/` and pipe it through stdin, so nothing is pasted into a shell string.
3. **Run** — pick a tool set per task: `--tools ""` for pure analysis, `Read,Glob,Grep` for read-only review, `Read,Glob,Grep,Edit,Write` for scoped edits.
4. **Verify** — parse the JSON output, check `is_error`, `subtype`, `result`, and the tool calls, then have the main agent inspect the actual changes.

### Command template

Pure analysis (no file access, no command execution):

```bash
codebuddy \
  --print \
  --model custom-local:BC-DeepSeek-V4.1-Flash \
  --tools "" \
  --output-format json \
  --max-turns 4 \
  --setting-sources user \
  --strict-mcp-config \
  --mcp-config '{"mcpServers":{}}' \
  --no-session-persistence \
  < work/codebuddy-prompt.txt > work/codebuddy-result.json
```

Read-only review: swap `--tools` for `--tools "Read,Glob,Grep" --allowedTools "Read,Glob,Grep" --permission-mode dontAsk`.

Scoped edits: run in a separate directory or a Git worktree with `--tools "Read,Glob,Grep,Edit,Write" --permission-mode acceptEdits`, and agree on the files that may change plus the definition of done.

### Flags

| Flag | Purpose |
| --- | --- |
| `--print` | Non-interactive mode; prints the result and exits |
| `--model custom-local:BC-DeepSeek-V4.1-Flash` | Pin the model without touching global defaults |
| `--tools ""` | Disable every tool; only reason over the prompt |
| `--output-format json` | Machine-parseable output |
| `--max-turns 4` | Bound turns to control latency and spend |
| `--setting-sources user` | Load user-level settings only |
| `--strict-mcp-config` + empty `--mcp-config` | Load no MCP servers, so nothing external is called by accident |
| `--no-session-persistence` | Keep no session on disk and leave history untouched |

### Caveats

- **Never trust the exit code alone.** The JSON output may be an event array or a single object; find the entry with `type: "result"` and check `is_error` and `subtype`.
- **Don't infer credit usage from `total_cost_usd`.** Actual billing is what your account statement says.
- **No inherited context.** CodeBuddy cannot see this conversation, so pass project constraints, file paths, and acceptance criteria explicitly.
- **Never let two delegates edit the same file.** Split parallel work into non-overlapping scopes.
- **Delegation grants no extra authority.** Publishing, deploying, remote changes, and outbound messages still need explicit approval.
- If login expires, credits run out, or the model is unavailable, report the real error — don't retry forever, silently swap models, or buy credits.

### License

[MIT](LICENSE) © 2026 zcx960
