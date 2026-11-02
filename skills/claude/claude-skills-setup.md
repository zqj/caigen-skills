# Setting up Caigen Skills for Claude Code

> This file is documentation, not a skill — it has no `SKILL.md`, so the client never loads
> it as one. It only explains how to get the four real skills (`caigen-auth`, `caigen-note`,
> `caigen-search`, `caigen-tag`) in front of Claude Code.

Claude Code reads skills from `~/.claude/skills` (personal, available in every project) or
`<project>/.claude/skills` (project-scoped, checked in with the repo). The Caigen skills are
plain directories with a `SKILL.md`, so installing them is just a copy.

## Before you start

The skills drive the `caigen` CLI, so the CLI has to exist and be logged in:

```bash
npm install -g @caigen/cli
caigen auth login --api-key cg_live_xxxxxxxx
```

Get the key from **Caigen Notes → Settings → API Key** at https://www.gobiji.cn.

## Option A — install with the CLI, then copy (recommended)

`caigen setup` writes the skills to `~/.agents/skills`. Copy them across from there so you
always get the version that matches your CLI:

```bash
caigen setup

mkdir -p ~/.claude/skills
cp -R ~/.agents/skills/caigen-auth \
      ~/.agents/skills/caigen-note \
      ~/.agents/skills/caigen-search \
      ~/.agents/skills/caigen-tag \
      ~/.claude/skills/
```

Re-run the copy whenever you upgrade the CLI — it refreshes all four skills in one go.

## Option B — copy straight from this repository

If you have this repo checked out and just want the skill files:

```bash
mkdir -p ~/.claude/skills
cp -R skills/caigen-auth skills/caigen-note skills/caigen-search skills/caigen-tag \
      ~/.claude/skills/
```

Run the same command with a project directory to keep the skills inside a single repo:

```bash
mkdir -p .claude/skills
cp -R skills/caigen-auth skills/caigen-note skills/caigen-search skills/caigen-tag \
      .claude/skills/
```

## Verify

```bash
ls ~/.claude/skills
```

You should see four directories, each containing a `SKILL.md`. Then restart Claude Code (or
start a new session) so the skills are discovered, and ask for something that triggers one —
*"show my recent notes"*, *"search my notes about MCP"*, *"list my tags"*.

To confirm the CLI side is healthy:

```bash
caigen auth status -o json
caigen notes --size 1 -o json
```

Both should print an envelope with `"success": true`.

## Notes

- Claude Code's personal directory (`~/.claude/skills`) applies everywhere; the project
  directory (`<project>/.claude/skills`) applies only inside that repo and is usually the
  better choice for a shared project.
- Skills never contain your API key. The key lives in `~/.config/caigen/config.json`, or is
  injected via `CAIGEN_API_KEY` in CI.
- Other clients that read `~/.agents/skills` need no copy at all — `caigen setup` is enough.
  See the [repository README](../../README.md) for the full picture.
