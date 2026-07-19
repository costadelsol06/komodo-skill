# komodo-skill

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) documenting
real, hard-won operational knowledge about [Komodo](https://komo.do) — a self-hosted
Docker/Compose orchestration platform (Core + Periphery architecture).

Every point in this skill was confirmed either by reading Komodo's own Rust source
directly or by reproducing the behavior on a live instance — not guessed from the docs
site alone, which lags behind and omits some details.

## What it covers

- Adopting an already-running `docker compose` project into a Komodo Stack without downtime
- The `auto_pull` trap: why deploying a locally-built (no-registry) image fails with
  `pull access denied`, and the correct per-service fix (`pull_policy: build`)
- The per-Stack deploy lock (`"Resource is busy"`) and how to sequence Procedures safely
- Why `Delete` on a Stack is **not** a safe metadata-only operation, and the
  non-destructive alternative (`RefreshStackCache`)
- Building images locally with no registry, and automating rebuild+redeploy on a schedule
- Periphery's networking requirements for `RunBuild` (DNS/egress gotchas), including
  enabling IPv6 on an isolated egress network without breaking isolation
- Why a scheduled Procedure can fail daily with a DNS/registry-auth-looking error that's
  actually a timezone-driven schedule collision with another job (e.g. a nightly backup)
- Scripting the Komodo API/CLI instead of clicking through the UI

## Install

This repo follows the [Agent Skills specification](https://agentskills.io/specification)
and works with the [`skills` CLI](https://github.com/vercel-labs/skills), compatible with
30+ agents/IDEs (Claude Code, Codex CLI, Cursor, VS Code, Gemini CLI, GitHub Copilot,
Goose, Roo Code, and more):

```bash
npx skills add costadelsol06/komodo-skill-v2
```

The CLI auto-detects which agents you have installed and places the skill in the right
directory for each (e.g. `~/.claude/skills/` or `.claude/skills/` for Claude Code).

**Manual install** (Claude Code): copy `skills/komodo/` into `~/.claude/skills/`:

```bash
cp -r skills/komodo ~/.claude/skills/
```

## License

MIT — use, adapt, and share freely.
