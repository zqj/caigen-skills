<div align="center">

# 菜根笔记 Skills

**面向 [菜根笔记](https://www.gobiji.cn) 的 Agent Skills —— AI 原生笔记。**

English docs: [README.md](README.md)

</div>

---

## 这是什么？

[菜根笔记](https://www.gobiji.cn) 是一个云端 AI 原生笔记产品。本仓库把它的能力打包成
Agent Skills：带 YAML frontmatter 的 Markdown 文件，任何兼容的 Agent 客户端都能加载并遵循。

每个 skill 都是对应功能的一页操作手册。Agent 读完 skill 后去驱动官方 `caigen` CLI ——
所有动作都走同一条已鉴权、可追溯的路径，而不是各自拼 HTTP 请求。

## 包含哪些 skill

| Skill | 一句话说明 |
| ----- | ---------- |
| [`caigen-note`](skills/caigen-note/SKILL.md) | 保存笔记、列出笔记、读取笔记全文。 |
| [`caigen-search`](skills/caigen-search/SKILL.md) | 搜索笔记内容，支持关键词与语义两种模式并处理分页。 |
| [`caigen-tag`](skills/caigen-tag/SKILL.md) | 查看全部标签、给笔记加标签、从笔记移除标签。 |
| [`caigen-auth`](skills/caigen-auth/SKILL.md) | 安装 CLI、配置 API Key、查看与清除登录状态。 |

## 环境要求

- **Node.js >= 18**（CLI 通过 npm 分发）。
- **菜根笔记账号** 和 **API Key**。
- **`caigen` CLI**（`@caigen/cli`）—— 它随包分发 skills，也负责真正干活。

## 快速开始

### 1. 安装 CLI

```bash
npm install -g @caigen/cli
```

CLI 以 `@caigen/cli` 发布，当前版本 0.1.0，正在向 npm 推送 —— 如果 npm 提示找不到包，
说明发布还没完成。`caigen-auth` skill 会完整跑通安装与验收流程，两种情况都适用。

### 2. 用 API Key 登录

在 https://www.gobiji.cn 的「菜根笔记 → 设置 → API Key」里取 Key，然后：

```bash
caigen auth login --api-key cg_live_xxxxxxxx
```

Key 的形态是 `cg_live_` 加 32 位十六进制字符。它会写入 `~/.config/caigen/config.json`
（仅本人可读写），CLI 永远不会完整回显它。

### 3. 安装 skills

```bash
caigen setup
```

这一步把 `caigen-*` skills 复制到 `~/.agents/skills`。**随后请重启或重载你的 Agent
客户端**，让它重新扫描 skills。

### 4. 验证连通

```bash
caigen notes --size 1 -o json
```

返回里 `"success": true`，说明 CLI 已安装、Key 有效、服务可达。之后直接用自然语言跟
Agent 说话就行 ——「记一下 X」「搜一下我写过的 Y」「给这条笔记打个 #阅读 标签」。

## 在各种 Agent 客户端里使用

这些 skill 是标准的 Agent Skills：一个包含 `SKILL.md`（带 `name` 与 `description`
frontmatter）的目录。任何支持该规范的客户端的发现方式都一样 —— 指向一个 skills 目录。

- **Hermes 及其它读取 `~/.agents/skills` 的客户端** —— 执行第 3 步的 `caigen setup`，然后
  重启会话。skills 会落在 `~/.agents/skills`，客户端自动加载。
- **Claude Code** —— 把 4 个 skill 目录复制到 `~/.claude/skills`（个人全局）或
  `<项目>/.claude/skills`（项目级）。完整步骤见
  [`skills/claude/claude-skills-setup.md`](skills/claude/claude-skills-setup.md)。
- **其它客户端** —— 把 `skills/` 下的 4 个目录复制到你的客户端读取 skills 的目录，或直接
  加载。

无论哪种方式，CLI 本身都必须已安装并完成登录：skills 教 Agent「怎么驱动 `caigen`」，
它不是 CLI 的替代品。

## 配置

| 环境变量 | 作用 | 默认值 |
| -------- | ---- | ------ |
| `CAIGEN_API_KEY` | 直接注入 API Key，优先级高于配置文件 —— CI 里常用，且不在磁盘留明文。 | — |
| `CAIGEN_BASE_URL` | 覆盖服务地址，`--base-url` 参数优先级高于它。 | `https://api.gobiji.cn` |
| `CAIGEN_CONFIG_DIR` | 覆盖配置目录。 | `~/.config/caigen` |

API Key 的取值顺序是：`CAIGEN_API_KEY` → `caigen auth login` 写入的配置文件。服务地址的
取值顺序是：`--base-url` → `CAIGEN_BASE_URL` → 配置文件 → `https://api.gobiji.cn`。

## 返回契约

所有命令在 stdout 输出同一个信封，Agent 可以确定性地解析：

```json
{
  "success": true,
  "data": {},
  "error": null,
  "request_id": "0000mvggpj4ionylt0kn"
}
```

失败时 `data` 为 `null`，`error` 里带 `code`、`message`、`retryable`。退出码是最快的判断
依据：`0` 成功、`1` 业务错误、`2` 配置错误、`3` 网络或超时、`4` 鉴权失败。机器调用加
`-o json`（`--json` 是别名）；默认的 `table` 输出是给人看的。

## 与 MCP Server 的关系

菜根笔记 MCP Server 端点为 https://b.gobiji.cn/mcp，通过 HTTP 暴露 12 个工具，与 CLI 互补：

- **CLI**（`@caigen/cli`）面向在终端里干活的 Agent —— 一个意图一条命令、人读输出、退出码，
  以及落在磁盘上的配置文件。
- **MCP Server** 面向需要直接调用结构化工具集的客户端。

两者操作的是同一个账号、同一份云端数据，按你的客户端选一个，或者两个都用。

## 仓库结构

    skills/
      caigen-auth/SKILL.md          安装、鉴权、验收
      caigen-note/SKILL.md          保存、列出、读取
      caigen-search/SKILL.md        关键词 + 语义搜索
      caigen-tag/SKILL.md           标签的列出、增加、移除
      claude/claude-skills-setup.md Claude Code 安装指引

## 开源许可

MIT，详见 [LICENSE](LICENSE)。
