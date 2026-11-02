---
name: caigen-note
description: 保存笔记、列出笔记、读取笔记全文。用户说“记一下/帮我记/新建笔记”“加个 todo/待办”“看看我的笔记/最近写了什么/有哪些笔记”“打开第 X 条/看看那条笔记写了什么”时使用。
---

# 菜根笔记写入、列表与全文读取

负责菜根笔记的三件事：记一条新笔记、按条件列笔记、按 ID 读全文。所有动作只通过 `caigen` CLI 执行。

## 意图路由表

| 意图                     | 命令                               |
| ------------------------ | ---------------------------------- |
| 记一条笔记 / 记个闪念    | `caigen save "<content>"`          |
| 记一条正式笔记           | `caigen save "<content>" --type 1` |
| 记一条待办               | `caigen save "<content>" --type 2` |
| 列出笔记                 | `caigen notes [选项] -o json`      |
| 读某条笔记全文           | `caigen note <id>`                 |
| 搜索笔记（不是本 Skill） | 交给 `caigen-search` skill         |

参数不确定时执行 `caigen --help` 读取真实用法，不要编造不存在的选项。

## 各命令的入参口径

- `caigen save <content> [--type 0|1|2]`
  - `type`：`0` = 闪念（默认）、`1` = 笔记、`2` = todo。用户说「随手记一下/闪念」用 0；「写成一篇笔记」用 1；「记个待办/要做的事」用 2。
  - `content` 是位置参数；含空格或中文标点时必须加引号，避免被 shell 拆成多个参数。
- `caigen notes [--page N] [--size N] [--order-by asc|desc] [--type -1|0|1|2] [--tag-id ID] [--recycle|--no-recycle]`
  - 默认：`page=1`、`size=20`、`orderBy=desc`、`type=-1`（全部）、`isRecycle=false`。
  - `--size` 上限 100；`--order-by asc` 按时间正序（最早在前），`desc`（默认）最新在前。
  - `--tag-id` 按标签 ID 过滤；标签 ID 可从 `caigen tag list -o json` 拿到。
  - `--recycle` 看回收站，默认 `false`（不看）。
  - 结果是分页的；要拿全量必须循环 `--page`，不要把单页当全量。
- `caigen note <id>`
  - `id` 必须是**正整数**；非正整数会直接被 CLI 拦下报 `INVALID_ARGUMENT`（退出码 1），不会发到服务端。

## 明确禁止

- 不要自己拼 HTTP 请求，不要直接调 tRPC / MCP 接口 —— `caigen` CLI 是唯一执行源。
- 不要猜笔记 ID。ID 只能来自 `caigen save` 的成功输出或 `caigen notes` / `caigen search` 的结果。
- 不要把 `caigen notes` 的列表当搜索结果：它是按时间/标签筛选的列表，没有相关性排序；要找内容请走 `caigen-search` skill。
- 写入类动作（`save`）会真实改动用户数据。用户只是问「有哪些笔记」时不要顺手写一条测试笔记。

## 统一结果契约

所有命令返回同一个信封：

```json
{ "success": true, "data": {}, "error": null, "request_id": "20位小写base36" }
```

- `success=false` 即失败；`error.code`、`error.message`、`error.retryable` 要读出来回给用户。
- `request_id` 用于报障，失败时一定要带上。
- **退出码第一判断**：0 成功 / 1 业务错误 / 2 配置错误 / 3 网络或超时 / 4 鉴权失败。**HTTP 200 或「命令跑过」不等于成功。**
- 机器调用统一加 `-o json`（`--json` 是别名）；table 是人读默认。
- 列表结果的 `data` 是分页对象（含 `list` 数组）；`note` / `save` 的 `data` 是单条笔记对象（含 `id`、`content`、`type`、时间字段）。

## 常见错误处置

| 情形                   | 错误码             | 退出码 | retryable | 处置                                               |
| ---------------------- | ------------------ | ------ | --------- | -------------------------------------------------- |
| 未登录 / Key 未配置    | `CONFIG_ERROR`     | 2      | false     | 转 `caigen-auth` skill 走 `auth login`；不要重试   |
| Key 失效或被吊销       | `UNAUTHORIZED`     | 4      | false     | 让用户重新获取 Key 并重新 `auth login`；重试无意义 |
| 服务不可达 / 5xx / 429 | `SERVER_ERROR`     | 3      | true      | 稍等重试，最多 2 次；仍失败则报告 `request_id`     |
| 超时                   | `TIMEOUT`          | 3      | true      | 同上，可考虑调大 `--timeout`                       |
| 笔记不存在             | `NOTE_NOT_FOUND`   | 1      | false     | 核对 ID；不要反复重试                              |
| 参数非法               | `INVALID_ARGUMENT` | 1      | false     | 修正参数后重跑                                     |

## 安全与恢复

- 不展示、不复述、不落日志完整 API Key；所有回显只走掩码（前 16 字符）。
- 不编造不存在的命令或参数；参数不确定就执行 `caigen --help`。
- `save` 成功只代表「已保存 #<id>」，不要进一步声称已同步到云端其它视图。
- 内容里含用户隐私时，列表输出只给必要字段的长预览，不要整篇回灌到聊天里。
- 失败时保留执行步骤、错误原因和 `request_id`；不要只回复「操作失败」。
