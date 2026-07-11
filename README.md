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
- Periphery's networking requirements for `RunBuild` (DNS/egress gotchas)
- Scripting the Komodo API/CLI instead of clicking through the UI

## Install

Copy `komodo/` into your Claude Code skills directory:

```bash
cp -r komodo ~/.claude/skills/
```

Or, for a project-local skill, into `.claude/skills/` at your repo root.

## License

MIT — use, adapt, and share freely.
