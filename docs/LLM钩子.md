### 引言

AI 虽然读取了指令文件（如 CLAUDE.md），但在实际执行时仍会忽略或违反这些规则

可以通过 “Hooks（钩子）” 机制来强制 AI 代理遵守特定指令

### 背景

开发者通常会在项目根目录下创建 `CLAUDE.md` 或 `AGENTS.md` 文件，用来告诉 AI：

> “当你运行测试时，请使用 `uv run pytest tests`，而不是直接运行 `pytest tests/`”

但实践中，AI 经常“忘记”这条规则，仍然输出：

```bash
$ pytest tests/
# command not found: pytest
```

因为 `pytest` 没有全局安装（而是通过 `uv` 工具管理），导致命令失败。用户不得不反复纠正 AI，体验极差

### 解决方案

在 AI 执行工具（如 Bash 命令）之前，插入一个检查钩子（Hook），自动拦截并拒绝不符合规范的操作

1. 配置 Hook 触发点 
   在 `~/.claude/settings.json` 中定义一个 `PreToolUse` 钩子，表示“在 AI 使用任何工具（如 Bash）之前先运行这个检查”

   ```json
   {
     "hooks": {
       "PreToolUse": [
         {
           "matcher": "Bash",  // 只对 Bash 命令生效
           "hooks": [
             {
               "type": "command",
               "command": "python ~/.claude/hooks/check-uv-pytest.py"
             }
           ]
         }
       ]
     }
   }
   ```

2. 编写检查脚本 
   创建脚本 `check-uv-pytest.py`，它会：

   - 从标准输入（stdin）读取 AI 即将执行的命令；
   - 如果命令包含 `pytest` 但没有 `uv run`，就报错并退出码为 `2`（非零表示失败）；
   - AI 看到这个错误后，会意识到命令不合法，从而重新生成正确的命令

   ```python
   #!/usr/bin/env python3
   import json
   import sys
   
   data = json.load(sys.stdin)
   cmd = data.get("tool_input", {}).get("command", "")
   
   if "pytest" in cmd and "uv run" not in cmd:
       print("Use 'uv run pytest' instead of bare 'pytest'", file=sys.stderr)
       sys.exit(2)  # 非零退出码 → 阻止命令执行
   ```

3. 效果 
   当 AI 试图运行 `pytest tests/` 时，钩子脚本会立即中断该操作，并返回错误提示。AI 会据此自我修正，最终输出正确的 `uv run pytest tests`

