---
name: caigen-tag
description: 查看菜根笔记的全部标签、给笔记加标签、从笔记移除标签。用户说“我有哪些标签/标签列表”“给它加个 #读书/打个标签”“把 #临时 去掉/取消这个标签”“这条笔记是哪些分类”时使用。
---

# 菜根笔记标签管理

负责标签的三件事：列出全部标签、给某条笔记加标签、从某条笔记移除标签。标签在菜根笔记里是**正文里的 hashtag**，不是独立的分类对象。

## 意图路由表

| 意图                                         | 命令                                                 |
| -------------------------------------------- | ---------------------------------------------------- |
| 列出全部标签                                 | `caigen tag list -o json`                            |
| 给笔记加标签                                 | `caigen tag add <note_id> <tag...>`                  |
| 从笔记移除标签                               | `caigen tag remove <note_id> <tag...>`               |
| 拿到标签 ID（供 `caigen notes --tag-id` 用） | 先 `caigen tag list -o json`，再 `caigen-note` skill |
| 读某条笔记的正文（看它现在有哪些标签）       | 交给 `caigen-note` skill 用 `caigen note <id>`       |

参数不确定时执行 `caigen --help` 读取真实用法，不要编造不存在的选项。

## 各命令的入参口径

- `caigen tag list -o json`
  - 列出账号下的**所有**标签，每项含 `name`、`id`、`type`。
  - 返回空数组时如实说「还没有标签」，不要因此说命令失败。
- `caigen tag add <note_id> <tag...>`
  - `note_id` 必须是**正整数**；标签可给多个（`caigen tag add 123 读书 笔记`）。
  - 标签可带 `#` 前缀也可不带（`#读书` 与 `读书` 等价），服务端会归一。
  - 返回 `{note, added, skipped}`：`added` 是真新增的，`skipped` 是正文里已经有的（重复添加不算错）。
- `caigen tag remove <note_id> <tag...>`
  - 同上，`note_id` 正整数、标签可多个、可带 `#`。
  - 返回 `{note, removed}`：`removed` 为空说明这条笔记上本来就没有那些标签。

## 关键口径（不要搞反）

- **标签名是字符串，不是 ID**。`add` / `remove` 都按**标签名**执行。
- **删除按标签名，不是按标签 ID**。CLI 没有「按 ID 删标签」的命令，不要编造 `tag delete <tag_id>` 之类。
- **`caigen tag list` 只有「列全部」一种形态**：CLI / MCP 没有「按笔记查标签」的命令，不存在 `caigen tag list <note_id>`。用户问「这条笔记有哪些标签」时，用 `caigen note <id>` 读正文，正文里的 `#xxx` 就是标签。
- 标签为空或只剩空白时，CLI 直接报 `INVALID_ARGUMENT`（退出码 1），不会发到服务端。

## 统一结果契约

所有命令返回同一个信封：

```json
{ "success": true, "data": {}, "error": null, "request_id": "20位小写base36" }
```

- `success=false` 即失败；`error.code`、`error.message`、`error.retryable` 要读出来回给用户。
- `request_id` 用于报障，失败时一定要带上。
- **退出码第一判断**：0 成功 / 1 业务错误 / 2 配置错误 / 3 网络或超时 / 4 鉴权失败。**HTTP 200 或「命令跑过」不等于成功。**
- 机器调用统一加 `-o json`（`--json` 是别名）；table 是人读默认。
- `tag list` 的 `data` 是标签数组；`tag add` / `tag remove` 的 `data` 是 `{note, added, removed, skipped}` 对象。

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
- `tag add` 会真实改写笔记正文。笔记 ID 不确定时先用 `caigen notes -o json` 查，不要猜。
- `skipped` 非空或 `removed` 为空都不是失败，如实说明「本来就有 / 本来就没有」即可，不要重试。
- 失败时保留执行步骤、错误原因和 `request_id`；不要只回复「操作失败」。
