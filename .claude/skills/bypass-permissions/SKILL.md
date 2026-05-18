---
name: bypass-permissions
description: >
  Toggle Claude Code permission mode ON/OFF. ON = all tool calls run without approval prompts.
  Asks scope (project-local or global) before acting. Supports revert via backup.
argument-hint: "[on|off|revert]"
allowed-tools: [Read, Write, Edit, Bash, Glob, AskUserQuestion]
disable-model-invocation: true
---

# Bypass Permissions

Toggle Claude Code's permission mode. When ON, all tool calls execute without user approval prompts.

## Phase 0: Confirm intent

Before ANY file change, use `AskUserQuestion` to confirm.

When action is **on**, ask TWO questions in sequence:

**Question 1 — Scope:**
- Question: "应用范围？"
- Options:
  - "仅当前项目" — write to `<project>/.claude/settings.local.json`
  - "全局（所有项目）" — write to `~/.claude/settings.json`
- multiSelect: false

**Question 2 — Confirm:**
- Question: "确认开启 bypassPermissions？工具调用将跳过审批弹窗。"
- Options: "确认开启" / "取消"
- multiSelect: false

When action is **off** or **revert**, ask ONE question:
- off: "确认关闭 bypassPermissions 恢复默认审批？" Options: "确认关闭" / "取消"
- revert: "确认回退到备份版本？" Options: "确认回退" / "取消"

If user selects "取消" for any question, STOP. Do not modify any files.

## Phase 1: Determine target path

| User selected | Target |
|---------------|--------|
| "仅当前项目" | `<project>/.claude/settings.local.json` |
| "全局（所有项目）" | `~/.claude/settings.json` |
| off (any scope) | Check which file exists and was modified; prefer project-local |
| revert | Check both `.bak` locations, use the one that exists |

If the target directory (`.claude/` or `~/.claude/`) does not exist, create it with `mkdir -p`.

## Phase 2: Backup

Before modifying the target file:
- If it exists, copy to `<target>.bak` (same directory). Overwrite existing `.bak`.
- If it doesn't exist, skip backup.

## Phase 3: ON

Read the target file if it exists. Start with `{}` otherwise.

Write/merge this `permissions` block:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(python *)",
      "Bash(python3 *)",
      "Bash(pip *)",
      "Bash(pip3 *)",
      "Bash(npm *)",
      "Bash(npx *)",
      "Bash(node *)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(ls *)",
      "Bash(cat *)",
      "Bash(mkdir *)",
      "Bash(rm *)",
      "Bash(cp *)",
      "Bash(mv *)",
      "Bash(find *)",
      "Bash(echo *)",
      "Bash(export *)",
      "Bash(which *)",
      "Bash(ollama *)",
      "Bash(docker *)",
      "Bash(sqlite3 *)",
      "Bash(powershell *)",
      "Bash(set *)",
      "Bash(start *)",
      "Bash(timeout *)",
      "Bash(dir *)",
      "Bash(del *)",
      "Bash(tasklist *)",
      "Bash(taskkill *)",
      "Bash(cmd *)",
      "Bash(cmd.exe *)",
      "Bash(setx *)",
      "Bash(where *)",
      "Bash(reg query *)",
      "Bash(powercfg *)",
      "Bash(PYTHONIOENCODING=utf-8 *)",
      "Read(**)"
    ],
    "defaultMode": "bypassPermissions",
    "skipDangerousModePermissionPrompt": true
  }
}
```

Merge rules:
- Preserve existing top-level keys (`hooks`, `env`, `model`, etc.).
- If `permissions.allow` already has entries, merge (union, no duplicates).
- Only overwrite `defaultMode` and `skipDangerousModePermissionPrompt`.

## Phase 3: OFF

1. Read target file.
2. Change `permissions.defaultMode` to `"default"`.
3. Set `permissions.skipDangerousModePermissionPrompt` to `false`.
4. Leave `permissions.allow` untouched (user may want to keep allow-list even in default mode).
5. Preserve all other keys.

## Phase 3: REVERT

1. Check `<target>.bak` exists. If not, also check the other scope's `.bak`.
2. If found, overwrite target with backup content.
3. If no backup anywhere: "没有找到备份文件，无法回退。可手动编辑 settings 文件。"
4. Report what was restored.

## Phase 4: After writing

1. Summarize changes (what keys changed, old → new).
2. Tell user: "已写入 <path>。输入 `/compact` 使当前会话即刻生效。"
3. Tell user how to undo:
   - "恢复默认：`/bypass-permissions off`"
   - "回退备份：`/bypass-permissions revert`"

## Notes

- The `allow` list is best-effort across Windows/Mac/Linux. Platform-specific commands (e.g., `taskkill`, `reg query`) are harmless on other platforms — they just never match.
- `disable-model-invocation: true` means this skill is NEVER auto-triggered. Only manual `/bypass-permissions` activates it.
- Project-local settings override global settings when both exist.
