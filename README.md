<div align="center">

# Caigen Skills

**Agent Skills for [Caigen Notes](https://www.gobiji.cn) — AI-native note-taking.**

让 AI 助手直接读你的菜根笔记。中文文档见 [README.zh-CN.md](README.zh-CN.md)。

</div>

---

## What is this?

[Caigen Notes](https://www.gobiji.cn) is a cloud, AI-native note-taking product. This
repository packages its agent capabilities as Agent Skills: plain Markdown files with YAML
frontmatter that any compatible agent client can load and follow.

Each skill is a short operating manual for one area of the product. The agent reads the
skill, then drives the official `caigen` CLI — so every action goes through a single,
authenticated, auditable path instead of ad-hoc HTTP calls.

## What's included

| Skill | What it does |
| ----- | ------------ |
| [`caigen-note`](skills/caigen-note/SKILL.md) | Save notes, list notes, and read a note's full text. |
| [`caigen-search`](skills/caigen-search/SKILL.md) | Search note content in keyword or semantic mode, with pagination. |
| [`caigen-tag`](skills/caigen-tag/SKILL.md) | List all tags, add tags to a note, and remove tags from a note. |
| [`caigen-auth`](skills/caigen-auth/SKILL.md) | Install the CLI, configure the API key, and check or clear login status. |

## Requirements

- **Node.js >= 18** (the CLI is distributed on npm).
- **A Caigen Notes account** and an **API Key**.
- The **`caigen` CLI** (`@caigen/cli`) — it ships the skills and does the actual work.

## Quick start

### 1. Install the CLI

```bash
npm install -g @caigen/cli
```

The CLI ships as `@caigen/cli` (v0.1.0) and is being rolled out to npm — if npm reports
the package as unavailable, it has not finished rolling out yet. The `caigen-auth` skill
covers the full install-and-verify loop either way.

### 2. Log in with your API key

Get your key from **Caigen Notes → Settings → API Key** at https://www.gobiji.cn, then:

```bash
caigen auth login --api-key cg_live_xxxxxxxx
```

Keys look like `cg_live_` followed by 32 hex characters. The key is stored in
`~/.config/caigen/config.json` with owner-only permissions; the CLI never prints it back
in full.

### 3. Install the skills

```bash
caigen setup
```

This copies the `caigen-*` skills into `~/.agents/skills`. **Restart or reload your agent
client afterwards** so it picks up the new skills.

### 4. Verify the connection

```bash
caigen notes --size 1 -o json
```

A JSON envelope with `"success": true` means the CLI is installed, the key is valid, and the
service is reachable. From here on, ask your agent in plain language — *"save a note about
X"*, *"search my notes about Y"*, *"tag this note #reading"*.

## Using the skills with your agent client

The skills are standard Agent Skills: a directory containing `SKILL.md` with `name` and
`description` frontmatter. Every client that supports the format discovers them the same
way — point it at a skills directory.

- **Hermes and other `~/.agents/skills` clients** — run `caigen setup` (step 3) and restart
  the session. The skills land in `~/.agents/skills` and are picked up automatically.
- **Claude Code** — copy the four skill directories into `~/.claude/skills` (personal) or
  `<project>/.claude/skills` (project-scoped). See
  [`skills/claude/claude-skills-setup.md`](skills/claude/claude-skills-setup.md) for a
  copy-paste walkthrough.
- **Anything else** — copy the four directories from `skills/` into whatever directory your
  client reads skills from, or load them directly.

In every case the CLI itself must still be installed and logged in: the skills teach the
agent *how* to drive `caigen`, they are not a replacement for it.

## Configuration

| Variable | Purpose | Default |
| -------- | ------- | ------- |
| `CAIGEN_API_KEY` | Inject the API key directly. Takes precedence over the stored config — useful in CI, and it keeps no plaintext key on disk. | — |
| `CAIGEN_BASE_URL` | Override the service base URL. The `--base-url` flag takes precedence over this. | `https://api.gobiji.cn` |
| `CAIGEN_CONFIG_DIR` | Override the configuration directory. | `~/.config/caigen` |

The API key is resolved in this order: `CAIGEN_API_KEY` → the config file written by
`caigen auth login`. The base URL is resolved in this order: `--base-url` →
`CAIGEN_BASE_URL` → the config file → `https://api.gobiji.cn`.

## Output contract

Every command prints the same envelope on stdout, so an agent can parse the result
deterministically:

```json
{
  "success": true,
  "data": {},
  "error": null,
  "request_id": "0000mvggpj4ionylt0kn"
}
```

On failure `data` is `null` and `error` carries `code`, `message`, and `retryable`. The
exit code is the fastest signal: `0` success, `1` business error, `2` configuration error,
`3` network or timeout, `4` authentication failure. Add `-o json` (`--json` is an alias)
for machine-readable output; the default `table` output is meant for humans.

## MCP Server

The Caigen Notes MCP Server is available at https://b.gobiji.cn/mcp and exposes 12 tools
over HTTP. It is complementary to the CLI:

- **CLI** (`@caigen/cli`) is for agents working in a terminal — one command per intent,
  human-readable output, exit codes, and a config file on disk.
- **MCP Server** is for clients that want a structured tool set to call directly.

Both surfaces act on the same account and the same cloud data, so pick whichever fits your
client — or use both.

## Repository layout

    skills/
      caigen-auth/SKILL.md          install, authenticate, verify
      caigen-note/SKILL.md          save, list, read
      caigen-search/SKILL.md        keyword + semantic search
      caigen-tag/SKILL.md           list, add, remove tags
      claude/claude-skills-setup.md Claude Code setup walkthrough

## License

MIT — see [LICENSE](LICENSE).
