---
name: caigen-search
description: 搜索菜根笔记内容，支持关键词与语义两种模式并处理分页。用户说“搜一下/找一下/查一下我写过什么”“搜关键词 X”“语义搜索/找相似的内容”“我关于某个话题记过什么”时使用。
---

# 菜根笔记搜索

负责按用户意图找到候选笔记，并给出「这是哪些笔记、可能在哪」的可核查答案。搜索只是第一步，读全文交给 `caigen-note` skill。

## 意图路由表

| 意图                     | 命令                                              |
| ------------------------ | ------------------------------------------------- |
| 关键词搜索               | `caigen search "<query>" --mode keyword -o json`  |
| 语义搜索（找意思相近的） | `caigen search "<query>" --mode semantic -o json` |
| 按类型过滤               | 加 `--type -1\|0\|1\|2`                           |
| 取下一页                 | 加 `--page 2`                                     |

参数不确定时执行 `caigen --help` 读取真实用法，不要编造不存在的选项。

## 各命令的入参口径

`caigen search <query> [--mode keyword|semantic] [--page N] [--size N] [--type -1|0|1|2]`

- `query` 是位置参数；含空格时必须加引号。
- 默认值：`mode=keyword`、`page=1`、`size=20`、`type=-1`（全部类型）。
- `--mode`：`keyword` 走字面匹配；`semantic` 走向量语义匹配，用户说「意思相近」「类似」「有没有讲过类似的」时用 `semantic`。
- `--type`：`0` 闪念 / `1` 笔记 / `2` todo / `-1` 全部。
- `--size` 上限 100；`--page` 必须 `>=1`，否则 CLI 直接报 `INVALID_ARGUMENT`（退出码 1）。

## 关键判断纪律

- **零命中不代表库里没有**：可能是同义词差异或措辞不同。此时换一个更短的关键词、或改用 `--mode semantic` 再试一次；仍为零才如实说「没搜到」，并建议用户改关键词或用 `caigen notes` 浏览。
- **分页不要假设单页全量**：结果被 `--size`（上限 100）截断。用户明确说「所有相关的」时，用 `--page 1,2,3...` 循环翻到空页或翻够用户要的条数为止。
- **不要把列表当详情**：`search` 只返回摘要（ID、类型、时间、内容预览）。找到候选后交给 `caigen-note` skill 用 `caigen note <id>` 读全文，再向用户复述内容。

## 统一结果契约

所有命令返回同一个信封：

```json
{ "success": true, "data": {}, "error": null, "request_id": "20位小写base36" }
```

- `success=false` 即失败；`error.code`、`error.message`、`error.retryable` 要读出来回给用户。
- `request_id` 用于报障，失败时一定要带上。
- **退出码第一判断**：0 成功 / 1 业务错误 / 2 配置错误 / 3 网络或超时 / 4 鉴权失败。**HTTP 200 或「命令跑过」不等于成功。**
- 机器调用统一加 `-o json`（`--json` 是别名）；table 是人读默认。
- 搜索结果的 `data` 是分页对象（含 `list` 数组），每项含 `id`、`type`、时间字段与 `content` 预览。

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
- 不猜测笔记内容：只复述 `caigen note <id>` 实际读回来的文本。
- `semantic` 模式是「意思相近」，不是精确命中；转述时要如实标注这是语义近似结果。
- 失败时保留执行步骤、错误原因和 `request_id`；不要只回复「没搜到」。
