---
name: ask-codebuddy
description: Delegate self-contained subtasks to the locally installed CodeBuddy CLI — code analysis, review, approach comparison, and scoped edits. Use when work can be handed off to CodeBuddy to save context or use existing credits. 通过本机 CodeBuddy CLI 处理独立子任务。
---

# Ask CodeBuddy

把本机 CodeBuddy 当作辅助子代理：主代理负责拆分任务、核验输出和整合结果，CodeBuddy 只完成被指派的独立部分。简单任务直接做；适合独立分析、审查或局部实现的工作可以委派，不必每次询问。

Use the locally installed CodeBuddy CLI as a helper sub-agent. The main agent owns the task — it splits the work, checks the output, and integrates the result. Small tasks are done directly; analysis, review, and scoped implementation can be delegated.

## Prerequisites

- `codebuddy` (alias `cbc`) is installed and on `PATH`; a common location is `~/.local/bin/codebuddy`. Confirm with `codebuddy --version`.
- The machine is already logged in and the account has credits.
- Calls run non-interactively. Reuse the existing login and never read, print, or store credentials.

Do not change the user's login, default model, MCP servers, or global permissions.

## Model selection

Leave `--model` out to use the account's default model. To pin one for a single call, pass the exact identifier reported by `codebuddy models` (or the client UI):

```bash
--model <model-id>
```

Some custom endpoints use a prefixed identifier such as `custom-local:<name>`. Pass it verbatim; never substitute a similarly named built-in model. If the requested model is unavailable, report the actual error instead of silently switching.

A successful call does not prove how many credits were charged, and `total_cost_usd` is not a credit count. Check the account statement when the amount matters.

## Invocation

Write the task to a temp file under `work/` and pass it through stdin, so nothing is pasted into a shell string. Pure analysis (no file access, no command execution):

```bash
codebuddy \
  --print \
  --tools "" \
  --output-format json \
  --max-turns 4 \
  --setting-sources user \
  --strict-mcp-config \
  --mcp-config '{"mcpServers":{}}' \
  --no-session-persistence \
  < work/codebuddy-prompt.txt > work/codebuddy-result.json
```

Add `--model <model-id>` when a specific model is required. These flags apply only to this process: `--setting-sources user` keeps user-level settings, and an empty MCP config prevents unintended calls to connected external services.

## Delegating

Give each delegation a self-contained goal, the context it needs, absolute file paths, what it may touch, and the expected deliverable. CodeBuddy does not inherit this conversation's context.

- **Analyze given content only:** `--tools ""`, with everything needed inside the prompt file.
- **Read files or review:** replace the tools with `--tools "Read,Glob,Grep" --allowedTools "Read,Glob,Grep" --permission-mode dontAsk`, and name the files or directories it may read.
- **Scoped implementation:** prefer a separate working directory or Git worktree, with `--tools "Read,Glob,Grep,Edit,Write" --permission-mode acceptEdits`, and agree on which files may change plus the definition of done. The main agent reviews the diff and runs the necessary checks. If command execution is genuinely required, grant it separately under the current task's authority.

Independent calls may run in parallel, but never let two agents edit the same file. Do not resume existing sessions or alter model configuration, login, MCP, or global permissions. Delegation grants no extra authority for publishing, deploying, remote changes, or outbound messages.

## Handling results

Keep results under `work/`. The JSON output may be an event array or a single object; find the entry with `type: "result"` and check `is_error`, `subtype`, and `result` — never judge success from the exit code alone.

Tool calls may appear in the event stream. Verify what was actually read or changed before relying on the conclusion: analysis needs evidence, implementation needs a reviewed diff and tests.

Set a reasonable timeout and `--max-turns`. On failure, make limited corrections based on the explicit error; if login expired, credits ran out, or the model is unavailable, tell the user what actually happened — do not buy credits or retry indefinitely.
