# Claude Permissions

一键配置 Claude Code 权限模式，让工具调用不再弹审批窗。

## 解决的问题

Claude Code 默认每次执行 Shell 命令、读写文件都会弹出确认框。手动编辑 `.claude/settings.local.json` 配置 `bypassPermissions` 需要知道 JSON 结构和字段名，路径也容易搞错。

这个 skill 把配置过程封装成一句话指令。

## 安装

将 `.claude/skills/claude-permissions/` 放到项目根目录，或放到 `~/.claude/skills/` 全局生效。

```bash
# 项目级（克隆到项目的 .claude/skills/ 下）
git clone https://github.com/DanDingBenShao/claude-permissions.git /tmp/claude-permissions
cp -r /tmp/claude-permissions/.claude/skills/claude-permissions .claude/skills/

# 全局（所有项目可用）
cp -r /tmp/claude-permissions/.claude/skills/claude-permissions ~/.claude/skills/
```

重启 Claude Code 会话即可识别。

## 使用

```
/claude-permissions on      开启 bypass 模式
/claude-permissions off     关闭，恢复默认审批
/claude-permissions revert  回退到上次备份
```

开启时会依次询问两个问题：

1. **应用范围** — 仅当前项目 / 全局所有项目
2. **确认开启** — 确认 / 取消

两次都确认后才写入配置文件。

## 做了什么

开启时写入 `settings.json` 三样东西：

| 配置 | 效果 |
|------|------|
| `defaultMode: "bypassPermissions"` | 所有工具调用不再弹确认窗 |
| `skipDangerousModePermissionPrompt: true` | 跳过危险操作的二次警告 |
| `allow: [...]` 预置常用命令模式 | git / python / npm / docker / curl 等免审批 |

写入前自动备份原文件到 `.bak`，随时可用 `revert` 回退。

## 安全设计

- **先问再改** — 每次操作前 `AskUserQuestion` 确认，选取消即停止
- **自动备份** — 写入前备份到 `.bak`，可随时回退
- **只改权限相关字段** — 已有 `hooks`、`env`、`model` 等配置保留不动
- **只增不删** — 已有 `allow` 列表只合并不覆盖
- **默认项目级** — 范围选择由用户决定，不替用户做选择

## 文件结构

```
.claude/skills/claude-permissions/
└── SKILL.md    # 技能指令（唯一文件）
```
