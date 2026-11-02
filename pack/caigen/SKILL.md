---
slug: caigen-notes
displayName: 菜根笔记
version: 0.1.1
summary: 菜根笔记——云端 AI 原生笔记。一套官方 Agent Skills 让你的 AI 助手直接记笔记、搜索、管标签、管登录，全部通过 caigen CLI 驱动。
license: MIT
category: 知识管理
icon: ./assets/icon.svg
name: caigen-notes
description: 菜根笔记官方技能包：让 AI 助手用自然语言直接操作你的云端笔记——记一下/帮我记/新建笔记/记个待办/看看我的笔记/搜一下我写过什么/语义搜索/加个标签/我有哪些标签/装一下 caigen/登录菜根笔记 等意图时使用。内含 4 个子 skill：caigen-note（保存/列出/读全文）、caigen-search（关键词+语义搜索）、caigen-tag（标签管理）、caigen-auth（安装 CLI 与 API Key 配置）。
---

# 菜根笔记官方技能包

一套 4 个子 skill，让你的 Agent 用自然语言直接操作菜根笔记（云端 AI 原生笔记，官网 https://www.gobiji.cn ）。所有动作只通过官方 `caigen` CLI 执行，不直接拼 HTTP。

## 前置（一次性）

1. 安装 CLI：`npm install -g @caigen/cli`
2. 取 API Key：菜根笔记网页端 → 设置 → API Key
3. 登录：`caigen auth login --api-key cg_live_xxxxxxxx`
4. 验证：`caigen notes --size 1 -o json` 返回 `"success": true` 即通

未登录时任何命令会报 `CONFIG_ERROR`（退出码 2），转 `skills/caigen-auth` 处理。

## 意图路由（按用户说法选子 skill）

| 用户说 | 子 skill | 命令 |
| --- | --- | --- |
| 记一下 / 帮我记 / 记个待办 | skills/caigen-note | `caigen save "<content>" [--type 0闪念\|1笔记\|2todo]` |
| 看看我的笔记 / 最近写了什么 | skills/caigen-note | `caigen notes [选项] -o json` |
| 打开第 X 条 / 读那条笔记 | skills/caigen-note | `caigen note <id>` |
| 搜一下 / 我写过什么关于 X | skills/caigen-search | `caigen search "<query>" --mode keyword\|semantic -o json` |
| 找意思相近的内容 | skills/caigen-search | `caigen search "<query>" --mode semantic -o json` |
| 加个标签 / 我有哪些标签 / 去掉标签 | skills/caigen-tag | `caigen tag list/add/remove ...` |
| 装 CLI / 配 Key / 看登录状态 | skills/caigen-auth | `caigen auth login/status/logout` |

**先读对应子 skill 的 SKILL.md 再执行**——每个子 skill 内含完整的入参口径、分页纪律、错误处置表和安全红线，本文件只做路由。

## 统一结果契约（所有命令共用）

```json
{ "success": true, "data": {}, "error": null, "request_id": "20位小写base36" }
```

- `success=false` 即失败；读 `error.code` / `error.message` / `error.retryable` 回给用户。
- 退出码第一判断：0 成功 / 1 业务错误 / 2 配置错误 / 3 网络或超时 / 4 鉴权失败。HTTP 200 不等于成功。
- 机器调用统一加 `-o json`；失败时必须带 `request_id` 上报，不要只说「操作失败」。

## 红线

- 不展示、不复述、不落日志完整 API Key；回显只走掩码（前 16 字符）。
- 不编造命令或参数；不确定就 `caigen --help`。
- 不猜笔记 ID；ID 只能来自 `save`/`notes`/`search` 的真实输出。
- 写入类动作（save）真实改动用户数据；用户只是查询时不要顺手写测试笔记。

## 子 skill 清单

- `skills/caigen-note/` — 保存、列出、读全文
- `skills/caigen-search/` — 关键词 + 语义双模式搜索与分页
- `skills/caigen-tag/` — 标签列表、加标签、移除标签
- `skills/caigen-auth/` — CLI 安装、API Key 配置、登录状态管理

## 源码与反馈

GitHub: https://github.com/zqj/caigen-skills （MIT）
