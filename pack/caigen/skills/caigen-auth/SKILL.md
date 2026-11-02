---
slug: caigen-auth
displayName: Caigen Auth
version: 0.1.0
summary: 安装 caigen CLI、配置 API Key、查看与清除菜根笔记登录状态。
license: MIT
name: caigen-auth
description: 安装菜根笔记 CLI、配置 API Key、查看与清除登录状态，把本机接入菜根笔记云端服务。用户说“安装/连接/登录/配置 Key”“没登录/提示未登录”“退出登录/登出”“看看登录状态”“装一下 caigen skill”时使用。
---

# 菜根笔记安装、连接与凭据管理

负责把“用户想用菜根笔记 CLI”推进到真正可执行的状态。不要只说“已安装”：CLI 可执行、Key 已配置、无写入验收通过三项都过了才算连接成功。

## 首次安装闭环

按顺序执行，已经满足的步骤直接跳过：

1. 用 `command -v caigen` 检查 CLI 是否存在；不要因为本 Skill 已安装就假设 CLI 存在。
2. CLI 缺失时自动执行 `npm install -g @caigen/cli`。这是 Agent 的工作，不要求用户手工复制命令。
3. 执行 `caigen auth login --api-key <用户提供的 Key>`。没有 Key 时不要编造，转到第 5 步的说明。
4. 执行 `caigen auth status -o json` 确认 `loggedIn=true`。
5. 执行 `caigen notes --size 1 -o json` 做**无写入验收**。用户同意前不要保存测试笔记。
6. 执行 `caigen setup -o json` 把 caigen-\* skill 装到 `~/.agents/skills`，并按 `next_actions[]` 提示用户重启或重载客户端会话。

## 意图路由表

| 意图             | 命令                                                    |
| ---------------- | ------------------------------------------------------- |
| 安装 CLI         | `npm install -g @caigen/cli`                            |
| 配置 API Key     | `caigen auth login --api-key <Key>`                     |
| 查看登录状态     | `caigen auth status -o json`                            |
| 清除已保存的 Key | `caigen auth logout`                                    |
| 安装 skill       | `caigen setup -o json`                                  |
| 无写入验收       | `caigen notes --size 1 -o json`                         |
| 切换服务地址     | `caigen <命令> --base-url <url>` 或设 `CAIGEN_BASE_URL` |

参数不确定时执行 `caigen --help` 读取真实用法，不要凭印象猜参数或编造不存在的选项。

## 凭据与配置口径

- **Key 形态**：`cg_live_` + 32 位十六进制。用正则 `^cg_live_[0-9a-f]{32}$` 自行校验一次，形态不合规就不要写入。
- **配置文件**：`~/.config/caigen/config.json`，目录权限 `0700`、文件权限 `0600`。写入是「先写临时文件再 rename」的原子操作。
- **落盘方式**：`auth login` 自己负责写入，不需要用户手工编辑文件。
- **Key 优先级**：环境变量 `CAIGEN_API_KEY` **高于**配置文件。CI / 临时试用可以直接注入环境变量，不留明文 key。
- **服务地址优先级**：`--base-url` > `CAIGEN_BASE_URL` > 配置文件 > 默认 `https://api.gobiji.cn`。
- **只回显掩码**：`auth status` 与 `auth login` 的成功输出里 `keyPrefix` 最多只有前 16 字符（`cg_live_` + 8 位 hex）。**绝不**展示、复述或记录完整 Key；用户主动贴出完整 Key 时，提醒其尽快在服务端吊销。

## 统一结果契约

所有命令返回同一个信封：

```json
{ "success": true, "data": {}, "error": null, "request_id": "20位小写base36" }
```

- `success=false` 即失败；`error.code`、`error.message`、`error.retryable` 要读出来回给用户。
- `request_id` 用于报障，失败时一定要带上。
- **退出码第一判断**：0 成功 / 1 业务错误 / 2 配置错误 / 3 网络或超时 / 4 鉴权失败。**HTTP 200 或「命令跑过」不等于成功。**
- 机器调用统一加 `-o json`（`--json` 是别名）；table 是人读默认。
- **CLI 是唯一执行源**：只执行 `caigen` 命令，禁止自己拼 HTTP 请求、禁止直接调 tRPC/MCP 接口。

## 常见错误处置

| 情形                   | 错误码             | 退出码 | retryable | 处置                                               |
| ---------------------- | ------------------ | ------ | --------- | -------------------------------------------------- |
| 未登录 / Key 未配置    | `CONFIG_ERROR`     | 2      | false     | 执行 `caigen auth login --api-key <Key>`；不要重试 |
| Key 失效或被吊销       | `UNAUTHORIZED`     | 4      | false     | 让用户重新获取 Key 并重新 `auth login`；重试无意义 |
| 服务不可达 / 5xx / 429 | `SERVER_ERROR`     | 3      | true      | 稍等重试，最多 2 次；仍失败则报告 `request_id`     |
| 超时                   | `TIMEOUT`          | 3      | true      | 同上，可考虑调大 `--timeout`                       |
| 笔记不存在             | `NOTE_NOT_FOUND`   | 1      | false     | 核对 ID；不要反复重试                              |
| 参数非法               | `INVALID_ARGUMENT` | 1      | false     | 修正参数后重跑                                     |

## 安全与恢复

- 不展示、不复述、不落日志完整 API Key；所有回显只走掩码（前 16 字符）。
- 用户未明确要求时不执行 `auth logout`。
- `CONFIG_ERROR` 里区分「Key 形态不合法」「配置文件损坏」「未登录」三类原因，给出不同的下一步。
- 失败时保留执行步骤、错误原因和 `request_id`；不要只回复「连接失败」。
- 菜根笔记是云端服务，只对接 `https://api.gobiji.cn`（或用户用 `--base-url` / `CAIGEN_BASE_URL` 显式指定的地址）。
