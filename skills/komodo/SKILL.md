---
name: komodo
description: Use when deploying, configuring, or troubleshooting Komodo (komo.do) — a Docker/Compose orchestration platform with Core/Periphery architecture. Covers Stack and Build resources, auto-update, custom image rebuilds via Procedures, the Komodo API/CLI, and specific failure modes like "pull access denied", "Resource is busy", stale cache after Delete, and Periphery network/build issues.
---

# Komodo

## Overview

Komodo (komo.do) is a self-hosted control plane for Docker: a web dashboard plus
**Core** (API/UI server) + **Periphery** (lightweight agent per managed host, talks to
the Docker socket). It deploys `docker compose` projects as **Stacks**, builds images
from Dockerfiles as **Builds**, and orchestrates both via **Procedures** on a
**Schedule**. It is explicitly *not* a container runtime — it drives the same
`docker`/`docker compose` CLI you'd run by hand, over SSH-like agent calls.

This skill documents behavior that is easy to get wrong even after reading the official
docs shallowly — all points below were confirmed either by reading Komodo's Rust source
directly or by reproducing the failure on a live instance.

## When to Use

- Standing up a new Komodo instance (Core + Periphery + Mongo/FerretDB) via Docker Compose
- Adopting an **already-running** `docker compose` project into a Komodo Stack
- A Stack shows a stale/wrong state (phantom services, "unhealthy" that doesn't match
  reality) after editing the underlying compose file
- A custom-built image (no registry, `build:` + local `image:` tag) won't deploy via
  Komodo — errors like `pull access denied`, `repository does not exist`
- A Procedure with parallel executions fails with `"Resource is busy"`
- Automating "rebuild a locally-built image, then redeploy just that service"
- Calling the Komodo HTTP API or `km` CLI programmatically
- Periphery's Docker builds fail with DNS/network errors despite the host having internet

## Quick Reference

| Symptom / Goal | Cause | Fix |
|---|---|---|
| `pull access denied`, `repository does not exist` on deploy | Stack `auto_pull` (default true) forces `docker compose pull <service>` before every deploy; fails for locally-built-only images | Add `pull_policy: build` to that **service** in the compose file (not the Stack setting) |
| `"Resource is busy"` executing two deploys in a Procedure stage | Two `DeployStack` calls targeting the **same Stack** ran in parallel — Komodo locks per-Stack | Never parallelize two deploys on the same Stack in one Procedure stage; different Stacks in parallel is fine |
| Stack shows wrong/stale service list ("unhealthy", phantom service) after editing compose | `info.latest_services` is a cache, only recomputed on Stack create/save via API/UI — not on restart, not on poll | Use `RefreshStackCache` (or delete+recreate as last resort — see Danger below) |
| Need to build a Docker image Komodo doesn't need to push anywhere | Registry is optional | Leave `image_registry` empty on the Build — image is built and tagged locally only |
| Need a periodic "rebuild custom image + redeploy" | `RunBuild` never triggers a Stack redeploy by itself | Procedure: Stage 1 `RunBuild` (parallel OK), Stage 2+ `DeployStack{services:[...]}` (one Stack per parallel group) |
| Periphery `docker build` fails resolving DNS for the base image | Periphery's own container network has no internet egress | Give Periphery's container a network with real internet egress — **not** the same bridge as public-facing containers (see Networking) |
| Need to script Komodo config instead of clicking through the UI | — | Generate an API key via `km create api-key`, call the HTTP API directly (see API section) |

## Architecture

- **Core**: web UI + REST/WS API. Stores all config in MongoDB (or FerretDB).
- **Periphery**: stateless agent, one per managed host, mounts the Docker socket, runs
  the actual `docker`/`docker compose` commands. Authenticates to Core via an Ed25519
  keypair exchanged automatically on first boot if both share a Docker volume — no
  manual key copying needed for a single-host bundled deployment.
- Resources: `Server`, `Stack` (compose), `Deployment` (single `docker run` container),
  `Build` (image from Dockerfile), `Builder` (where a Build runs), `Procedure`
  (multi-stage automation), `ResourceSync` (declarative TOML-in-git for everything).

## Deploying an Existing Compose Project (adopt, don't migrate)

To attach Komodo to a compose project that's **already running**, without touching it:

1. Use Stack mode **"Files on Server"**, pointing `run_directory` + `file_paths` at the
   real, already-running directory on the host — not a fresh git clone. A git clone puts
   the files at a different path, breaking any relative bind mounts in the compose file.
2. Periphery only sees paths under its `root_directory` (default `/etc/komodo`) by
   default. For "Files on Server" pointing elsewhere, bind-mount that host directory
   into the Periphery container at the **same path** (Komodo requires host path ==
   container path for this).
3. **Set `project_name` explicitly**, matching the real Docker Compose project name
   exactly (case-sensitive). Verify with `docker compose ls` on the host first. A
   mismatch doesn't error — Komodo just can't find the "expected" containers and shows
   the Stack as down/unhealthy, since Docker Compose project identity is a label match,
   not a name-guessing heuristic.
4. Leave the Stack's `environment` field empty — Compose auto-loads the existing
   `.env` from `run_directory`. Don't duplicate secrets into Komodo's UI/DB.

## The `auto_pull` Trap (locally-built images)

Komodo's Stack config has `auto_pull` (default `true`). Every `DeployStack` execution
runs, in order:

```
docker compose pull <services>   # only if auto_pull=true; abort on failure
docker compose up -d <services>
```

For a service that's `build:`-only with **no real registry image** (e.g.
`image: my-custom-app:latest` where `my-custom-app` was never pushed anywhere), the
`pull` step always fails: `pull access denied for my-custom-app, repository does not
exist`. Komodo aborts the whole deploy at that point — **the running container is never
touched**, but the fresh image you just built is also never applied.

**Do not disable `auto_pull` at the Stack level** to fix this — it also gates the pull
step for every *other* service in the Stack, including registry-backed ones your
auto-update relies on.

**Correct fix**: set Docker Compose's own per-service `pull_policy: build` on just the
locally-built service(s):

```yaml
services:
  my-custom-app:
    build: ./my-custom-app
    image: my-custom-app:latest
    pull_policy: build   # tells Compose to never attempt a registry pull for this service
```

Verify directly before trusting it: `docker compose pull my-custom-app` should print
`Image my-custom-app:latest Skipped` instead of erroring.

A service with **only** `build:` and no `image:` field at all is naturally immune (there's
nothing for Compose to try pulling), but any service that also sets an explicit `image:`
tag needs `pull_policy: build` explicitly.

## Same-Stack Deploy Lock

Komodo serializes deploy-type actions **per Stack**. Two `DeployStack` executions
targeting the same Stack, dispatched in parallel (e.g. two executions in one Procedure
stage), race for a lock — the loser fails with `"Resource is busy"`. The one that lost
the race does not touch its target container; nothing is corrupted, but nothing is
applied either.

Executions against **different** Stacks run fine in parallel. Design Procedures
accordingly:

```toml
[[procedure.config.stage]]
name = "Deploy (different stacks, safe in parallel)"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["svc1"] } },
  { execution.type = "DeployStack", execution.params = { stack = "stack-b", services = ["svc2"] } },
]

[[procedure.config.stage]]
name = "Deploy (same stack as above — must be sequential)"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["svc3"] } },
]
```

**Two or more locally-built services in the same Stack** (both the `auto_pull` trap and
this lock apply together): give every one of them `pull_policy: build` (see previous
section), then prefer **one `DeployStack` call listing all of them** in `services` over
several separate calls — `services` accepts a list, and one call against a Stack never
races itself:

```toml
[[procedure.config.stage]]
name = "Deploy both locally-built services in one call — no lock risk"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["svc1", "svc2"] } },
]
```

Only fall back to sequential stages (one execution per stage) if you specifically need
them deployed one at a time rather than together.

## Stale Cache vs. Destructive Delete — read this before deleting anything

Komodo caches a Stack's expected container list in `info.latest_services`. This cache is
only recomputed when the Stack is **created or its config saved** through the normal
API/UI path — **not** by a Core restart, not by the periodic monitor, not by editing the
compose file on disk. If you remove a service from the compose file, the Stack keeps
believing it should exist, and shows the Stack as unhealthy/wrong indefinitely.

There is a dedicated, safe fix: the **`RefreshStackCache`** action (internally used by
Komodo's own update-check flow, `check_stack_for_update_inner`) recomputes this cache
**without deploying or touching any container**. Prefer this over anything destructive.
Same param shape as every other Stack action — one field, the Stack id or name:

```bash
# Reads credentials from env vars — never hardcode or print a key/secret (see API section)
curl -X POST https://<your-komodo-host>/write \
  -H "X-Api-Key: $KOMODO_API_KEY" -H "X-Api-Secret: $KOMODO_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"type":"RefreshStackCache","params":{"stack":"<stack-id-or-name>"}}'
```

**⚠️ `Delete` on a Stack is NOT purely a metadata operation.** Deleting a Stack (even in
"Files on Server" mode, where Komodo never wrote the compose file itself) can trigger a
real `docker compose down` on the underlying project — every container in that Stack
stops and is removed. Named volumes are preserved (`down` without `-v`), so data
survives, but **all containers in the Stack go offline simultaneously** until you run
`docker compose up -d` again from the untouched compose file. This is confirmed
behavior, not a bug — Komodo's own container-lifecycle docs describe "Remove" as
destroying the container, and `Delete` on a Stack follows the same philosophy. It is
**not reliably safe** to assume otherwise just because it worked without incident once.

**Rule of thumb:**
- Stale display / wrong cached service list → `RefreshStackCache`.
- Genuinely need to reconstruct a Stack resource → `Create` a fresh one (adopting
  already-running containers via "Files on Server" is safe and does not touch them).
- Avoid `Delete` on a Stack entirely unless you are prepared for every container in it
  to go down, and have a plan (and the compose file) to bring them back up immediately.

## Building Custom Images Locally (no registry needed)

A Build resource doesn't require a registry. Leave `image_registry` empty (`[]`) and
Komodo runs `docker build` on the configured Builder, tagging the result **locally only**
— nothing is pushed anywhere. This is the right setup when the Builder is the same host
that will also run the resulting container (no need to round-trip through a registry).

Three source modes exist: write the Dockerfile in the UI, clone a git repo, or
**"Files on host"** — build from a Dockerfile + context already present on the Builder,
same pattern as adopting an existing Stack. `files_on_host` avoids needing git
credentials in Komodo at all for images that are only ever built where they run.

`RunBuild` **never triggers a Stack redeploy by itself** — building and deploying are
separate concerns in Komodo (the one exception: a `Deployment`, singular-container
resource, explicitly configured with "Redeploy on Build"). To actually run the fresh
image, follow the build with an explicit `DeployStack{services:[...]}` targeting just
that service (see the deploy-lock section above for how to sequence multiple such calls
safely in a Procedure).

## Automating Rebuild + Redeploy on a Schedule

Combine a `Procedure` (multi-stage automation) with a `Schedule`:

```toml
[[procedure]]
name = "Rebuild custom images"
[procedure.config]
schedule_format = "English"        # or "Cron" — see below
schedule = "Every day at 04:00"
schedule_enabled = true
schedule_alert = false             # true = alert on every run, even success
failure_alert = true               # alert only when the procedure fails

[[procedure.config.stage]]
name = "Build"
executions = [
  { execution.type = "RunBuild", execution.params.build = "app-a" },
  { execution.type = "RunBuild", execution.params.build = "app-b" },
]

[[procedure.config.stage]]
name = "Deploy"
executions = [
  { execution.type = "DeployStack", execution.params = { stack = "stack-a", services = ["app-a"] } },
  { execution.type = "DeployStack", execution.params = { stack = "stack-b", services = ["app-b"] } },
]
```

Notes:
- `schedule_format = "English"` accepts phrases like `"Every day at 04:00"`,
  `"Every 5 minutes"`, `"At midnight on the 1st and 15th of the month"` — there is no
  native "every N weeks" expression in either English or Cron format; approximate it
  with two fixed days per month if needed.
- A daily cadence for image rebuilds is normally cheap even when nothing changed:
  Docker's build cache makes an unchanged base a near-instant no-op, and
  `docker compose up` is itself a no-op when the resulting image is byte-identical, so
  there's rarely a good reason to default to a sparser schedule "to be safe" — that
  instinct is usually solving a cost problem that doesn't actually exist here.
- A separate built-in Procedure, **"Global Auto Update"** (created automatically on
  install, daily), handles registry-backed images independently via `auto_update` /
  `poll_for_updates` on Stacks/Deployments — that mechanism compares digests **under the
  same tag already in the compose file** (never jumps to a different tag/version), and
  does not go through the same `auto_pull`-gated deploy path described above, so a
  Stack's `auto_pull` setting does not need to be touched to keep that mechanism working.
- Exclude locally-built services from a Stack's `auto_update_skip_services` list — they
  have no registry to poll, so leaving them out avoids meaningless check attempts (though
  it isn't unsafe if you don't).

## Networking Requirements for Periphery

Periphery needs the Docker socket (root-equivalent — same trust level as any tool doing
this, e.g. Watchtower/Portainer) and, if it will ever run `RunBuild`, **outbound internet
access on its own container network** — `docker build` invoked through Periphery resolves
DNS and fetches base-image layers using Periphery's own network namespace, not
transparently through the host's network the way `docker compose up` for already-cached
images does. An `internal: true` (no-egress) network on Periphery will make any build
fail with DNS errors while deploys of already-pulled images keep working fine — that
asymmetry is what makes this confusing to diagnose.

Give Periphery a **dedicated** network with real egress for this, rather than reusing a
bridge network shared with public-facing containers (reverse proxy, web apps). Periphery
holds the Docker socket; co-locating it on the same L2 segment as internet-facing
containers increases the blast radius if one of those is ever compromised, even though
Periphery authenticates inbound connections by keypair. A separate outbound-only bridge
network gives Periphery internet access with zero network-level reachability from (or to)
the public-facing containers.

## Using the API / CLI Instead of the UI

The `km` CLI ships inside the Core image. Run it via
`docker exec <core-container> km ...` — invoked this way it can read Core's own database
config directly, no separate credentials needed for that specific context.

To script the HTTP API (e.g. to create resources without hand-filling UI forms, which is
easy to mistype on multi-field forms like Procedure stages):

```bash
# One-time: mint a key for an existing user, straight from the DB (no browser/session needed).
# Store both fields directly into a chmod-600 file or env vars — never echo/print/log a
# key or secret value; treat them as write-only from the moment they're generated.
docker exec <core-container> km create api-key "my-automation" --for <username> \
  > /path/to/credentials-file   # chmod 600 this file immediately, before reading it
```

```bash
# Wire protocol confirmed empirically: POST to /read, /write, /execute with these headers.
# Source credentials from env vars (or the file above) — never inline a literal key/secret
# in a command, and never have the agent print/output a key or secret value verbatim.
curl -X POST https://<your-komodo-host>/write \
  -H "X-Api-Key: $KOMODO_API_KEY" \
  -H "X-Api-Secret: $KOMODO_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"type":"CreateProcedure","params":{"name":"...", "config": {...}}}'
```

Every write/execute action name (`CreateProcedure`, `RunProcedure`, `DeployStack`,
`RunBuild`, `UpdateProcedure`, ...) and its exact param shape is defined as a Rust struct
in `client/core/rs/src/api/{read,write,execute}/*.rs` in Komodo's own repo — reading the
struct definition directly is faster and more reliable than guessing from the docs site,
which lags behind and omits some fields (e.g. `EnabledExecution` wraps each execution as
`{"execution": {"type": ..., "params": {...}}, "enabled": true}` — not the flatter shape
the TOML sync examples might suggest at a glance).

Async executions (`RunProcedure`, `RunBuild`, `DeployStack`, ...) return immediately with
`status: "InProgress"` and an `_id`. Poll with `GetUpdate{id}` until `status: "Complete"`,
then check `success` — a top-level `success: true` on a multi-stage Procedure does not
guarantee every individual execution inside it succeeded silently; read the `logs[]`
array, and follow any `see update '<id>'` reference in an error trace to get the specific
failing execution's own detailed log.

## Common Mistakes

- **Assuming a Stack `Delete` is always safe because it was safe once.** Test in a
  low-stakes context, or avoid entirely — see the Danger section above.
- **Setting `pull_policy: build` at the Stack level.** It's a Docker Compose per-service
  field, not a Komodo Stack setting — it belongs in the compose YAML.
- **Parallelizing two deploys on the same Stack "for speed."** Costs you a confusing
  `"Resource is busy"` failure instead of a few seconds saved.
- **Trusting a compose service's apparent creation timestamp as proof a build actually
  ran.** Docker's content-addressable image store reuses the same image ID (and its
  original timestamp) when a fresh build produces byte-identical layers to a prior one —
  an old-looking timestamp after a successful `RunBuild` is normal, not evidence of
  failure. Verify content directly (e.g. run the binary and check its reported version)
  if you need certainty, not the image's `Created` field alone.
- **Reading only the top-level Procedure result.** Errors from one failed execution in a
  parallel stage can overshadow siblings that also failed; always drill into individual
  execution logs when a stage reports a failure.
