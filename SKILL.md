---
name: ask-codebuddy
description: 通过本机 CodeBuddy CLI 和用户已登录的账户处理独立子任务。适用于委派代码分析、审查、方案比较和局部实现，或用户要求用 CodeBuddy 积分辅助工作时。
---

# Ask CodeBuddy

把本机 CodeBuddy 当作辅助子代理使用，以便利用已有订阅或积分。主代理负责拆分任务、核验输出和整合结果。简单任务直接完成；适合独立分析、审查或局部实现的工作可以委派，无需每次询问。

## 本机调用

CLI 为 `codebuddy`，`cbc` 是同一工具的别名；若不在 `PATH` 中，常见安装位置是 `~/.local/bin/codebuddy`。调用前先用 `codebuddy --version` 确认可用。沿用现有登录，不读取或输出凭据。

推荐默认使用 `BC-DeepSeek-V4.1-Flash`，完整 CLI 标识为 `custom-local:BC-DeepSeek-V4.1-Flash`。每次调用显式指定模型，不修改 CodeBuddy 原有默认设置。这里的 `custom-local:` 前缀是该模型的实际标识，不应因此替换为内置模型 `hy3` 或 `deepseek-v4.1-flash`。用户指定其他模型时遵从；所选模型不可用时说明实际错误，不擅自换模型。调用成功不等于已确认积分扣费数额，不能用 `total_cost_usd` 推断积分消耗。

把任务写入工作区的 `work/` 临时文件，通过标准输入传递，避免把任务内容拼接进 shell。纯分析示例：

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

参数限制只影响该进程。`--setting-sources user` 保留用户设置，任务涉及的项目约束由主代理明确提供；空 MCP 配置避免无意调用用户连接的外部服务。

## 分配任务

给出独立目标、必要上下文、文件绝对路径、允许操作和预期交付物。需要提供项目约束时，传递当前任务适用的指令。CodeBuddy 不继承本对话的上下文。

- **仅分析给定内容：** 使用 `--tools ""`，把必要内容放进提示文件。
- **读取文件或审查：** 将工具参数替换为 `--tools "Read,Glob,Grep" --allowedTools "Read,Glob,Grep" --permission-mode dontAsk`。明确允许读取的文件或目录。
- **局部实现：** 优先在独立工作目录或 Git worktree 中运行，使用 `--tools "Read,Glob,Grep,Edit,Write" --permission-mode acceptEdits`，约定可修改的文件和完成条件。由主代理检查变更并运行必要验证。若确实需要命令执行，按当前任务授权范围单独选择工具和权限。

可独立的调用可以并行，但不要让多个代理同时修改相同文件。不要恢复用户现有会话，也不要擅自改变模型配置、登录、MCP 或全局权限。发布、部署、远端变更和对外发消息不因委派获得额外授权。

## 接收结果

保存结果在工作区 `work/`。JSON 输出实测可能是事件数组，也可能是单个对象；查找 `type: "result"` 的结果对象，检查 `is_error`、`subtype` 和 `result`，不要仅凭退出码判断成功。

工具调用可能包含在事件中。核对实际访问和改动，再使用代理结论。分析结果需要检查证据；实现结果需要检查差异和必要测试。

为调用设置合理的时限与 `--max-turns`。失败时根据明确错误做有限修正；遇到登录、积分不足或模型不可用时向用户说明实际情况，不自动购买积分或无限重试。
