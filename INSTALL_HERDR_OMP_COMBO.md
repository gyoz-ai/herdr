> Local, machine-specific setup notes for running herdr + OMP together from the gyoz-ai forks. Not upstream-facing (see `README.md` for the public install path); this doc documents what is actually wired up on this workstation, per `LOCAL_FORK.md` in both repos.

# 1. Overview

herdr is a terminal-native agent multiplexer (workspaces / tabs / panes) that owns a local unix-socket server. OMP (Oh My Pi) is the coding agent that runs inside a herdr-spawned pane. The "combo" is: herdr spawns a pane, injects a few env vars into it, OMP starts in that pane and loads a herdr-managed extension that reports its live agent state (`idle`/`working`/`blocked`) back to herdr over that socket — which is what powers herdr's sidebar agent status and `herdr wait agent-status`.

Both halves are gyoz-ai forks, one-feature-one-commit ledgers rebased onto their respective upstreams via the `fork-sync` skill:

| clone | branch | origin (push target) | upstream (never pushed) |
|---|---|---|---|
| `~/Projects/gyoz-ai/herdr` | `master` | `git@github.com:gyoz-ai/herdr.git` | `git@github.com:ogulcancelik/herdr.git` |
| `~/Projects/gyoz-ai/omp` | `main` | `git@github.com:gyoz-ai/oh-my-pi.git` | `git@github.com:can1357/oh-my-pi.git` (baseline v16.3.15) |

# 2. Prerequisites

- **Rust toolchain + cargo** — herdr is a Rust binary crate (`Cargo.toml`: `herdr` v0.7.3, `build = "build.rs"`).
- **`cargo-nextest`** — herdr's test runner (`cargo nextest run`).
- **`zig@0.15` via Homebrew, keg-only** — herdr's `build.rs` vendors `libghostty-vt` through zig; it must be put on `PATH` explicitly for every build:
  ```sh
  brew install zig@0.15
  ```
- **Bun (`bun@1.3.14`, per `packageManager` in `package.json`)** — OMP is a Bun/TypeScript monorepo; the fork's coding agent runs live from `src/` via Bun, no build step in the dev loop.
- **Docker** — only needed if you want the `memory` OMP extension (session memory backed by a local Typesense container). Not required for the herdr↔OMP bridge itself.

# 3. Install herdr from the gyoz-ai fork

```sh
cd ~/Projects/gyoz-ai/herdr
git remote -v   # confirm origin=gyoz-ai/herdr.git, upstream=ogulcancelik/herdr.git
```

Build (zig must be on `PATH` for the vendored `libghostty-vt` step):

```sh
PATH="$(brew --prefix zig@0.15)/bin:$PATH" cargo build --release --locked
```

Test (optional, before activating):

```sh
cargo nextest run
```

If a gpg-agent prompt hangs the worktree tests, disable signing for just that run instead of touching git config:

```sh
GIT_CONFIG_COUNT=2 GIT_CONFIG_KEY_0=commit.gpgsign GIT_CONFIG_VALUE_0=false GIT_CONFIG_KEY_1=tag.gpgsign GIT_CONFIG_VALUE_1=false cargo nextest run
```

**Activate the build.** The active binary on this machine is a plain-file copy at `/opt/homebrew/bin/herdr` that shadows the brew 0.7.3 formula — there is no symlink and no `cargo install`:

```sh
cp target/release/herdr /opt/homebrew/bin/herdr
```

then restart herdr (quit and relaunch the herdr process/daemon — closing just a client pane does not restart the daemon).

> **Do NOT run `brew link --overwrite herdr`.** That restores the upstream Homebrew bottle, overwriting the fork build you just activated.

# 4. Install OMP from the gyoz-ai fork

```sh
cd ~/Projects/gyoz-ai/omp
git remote -v   # confirm origin=gyoz-ai/oh-my-pi.git, upstream=can1357/oh-my-pi.git
```

From-scratch bootstrap (installs deps, builds native addons, links the `coding-agent` workspace package, then points the global `omp` binary at the fork source):

```sh
bun install
bun run build:native
bun --cwd=packages/coding-agent link
sh scripts/link-omp.sh
```

(the four steps above are exactly the repo's `"setup"` script in `package.json` — `bun install && bun run build:native && bun --cwd=packages/coding-agent link && sh scripts/link-omp.sh`.)

`scripts/link-omp.sh` replaces the raw bun-shebang symlink (which points at `src/cli.ts`) with the safer dev-launcher wrapper `packages/coding-agent/scripts/omp`, and symlinks Bun's global bin (`bun pm -g bin`, or `${BUN_INSTALL:-$HOME/.bun}/bin` as a fallback) to it — so `~/.bun/bin/omp` ends up running `exec bun --preload scripts/omp.ts ../src/cli.ts` against the fork's live source.

**Activating changes:** the fork's `src/` runs live — no bundle/build step in the loop. A source edit takes effect on the next `omp` restart; a running session keeps the old code. Re-run `bun install` only when `package.json` / `bun.lock` change. (`dist/cli.js`, built by `bun run gen:bundle`, only matters for the dormant npm-installed copy — the dev launcher never reads it.)

Gate before trusting a build (run from `packages/coding-agent`):

```sh
bun run check:types
GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_SYSTEM=/dev/null bun run test
```

# 5. Wire OMP ↔ herdr

The bridge is a single herdr-managed OMP extension: `~/.omp/agent/extensions/herdr-omp-agent-state.ts`. Its header is explicit about ownership:

```ts
// installed by herdr
// managed by herdr; reinstalling or updating the integration overwrites this file.
// add custom hooks/plugins beside this file instead of editing it.
// HERDR_INTEGRATION_ID=omp
// HERDR_INTEGRATION_VERSION=4
```

**What it does:** on OMP lifecycle events (`session_start`, `session_switch`, `agent_start`, `agent_end`, `tool_approval_requested`/`resolved`, the `ask` tool's execution start/end, `session_shutdown`) it opens a short-lived unix-socket connection and writes one newline-delimited JSON-RPC request per state change:
- `pane.report_agent_session` — announces this OMP session (id/path) to herdr for a given pane.
- `pane.report_agent` — reports `state: "working" | "blocked" | "idle"` (debounced into idle via `HERDR_OMP_IDLE_DEBOUNCE_MS`, default 250ms; held at "working" through transient provider errors for `HERDR_OMP_RETRY_GRACE_MS`, default 2500ms).
- `pane.release_agent` — releases herdr's authority over the pane, sent only on `session_shutdown` when `event.reason === "quit"` (a real process quit — never on `/reload`, `/new`, `/resume`, `/fork`).

Only the root chief session reports (`ctx.hasUI === true`); subagent sessions never talk to the socket.

**Env vars (all supplied by herdr itself, per-pane — never set these by hand):**
- `HERDR_ENV` — must be `"1"`.
- `HERDR_SOCKET_PATH` — filesystem path to herdr's unix socket for this pane; no fixed/default path, herdr assigns it dynamically per instance.
- `HERDR_PANE_ID` — which pane this OMP process's state reports belong to.
- `HERDR_OMP_IDLE_DEBOUNCE_MS` (optional, default `250`) and `HERDR_OMP_RETRY_GRACE_MS` (optional, default `2500`) — tuning knobs, settable in the pane's environment if you want to override the defaults.

The extension's gate is exactly:

```ts
function enabled() {
  return HERDR_ENV === "1" && !!socketPath && !!paneId;
}
```

If any of the three required vars is missing, every hook body short-circuits (`if (!enabled()) return;`) and the OMP session runs with zero socket traffic.

**Enabling it in `~/.omp/agent/config.yml`:** nothing to add. `config.yml` only configures `modelRoles` / `advisor` / `task.agentModelOverrides` — there is no extension enable-list. OMP auto-discovers every top-level file/dir with an `index.ts` (or, for a lone file like this one, the file itself) under `~/.omp/agent/extensions/` at startup and live-reloads on mtime change. Presence of `herdr-omp-agent-state.ts` in that directory *is* the registration; herdr's own installer/updater is what drops (and overwrites) this file — it is not something to hand-author or hand-edit. Add any custom hooks in a **new sibling file/dir** under `extensions/` instead of editing this one.

**Startup order:**
1. herdr must already be running (it owns the socket server and spawns panes).
2. Spawn/open a pane in herdr and start OMP inside it (e.g. run `omp` in that pane). herdr injects `HERDR_ENV=1`, `HERDR_SOCKET_PATH`, and `HERDR_PANE_ID` into the pane's environment before/at process launch.
3. OMP's extension loader auto-loads `herdr-omp-agent-state.ts` from `~/.omp/agent/extensions/` like any other extension; its `enabled()` gate passes because the env vars are present, and it starts reporting state.

Launching OMP from a bare terminal (outside a herdr pane) is harmless but inert: the extension loads, `enabled()` is false, and it never touches the socket.

# 6. Verify

From inside a herdr pane running OMP:

```sh
echo "$HERDR_ENV $HERDR_SOCKET_PATH $HERDR_PANE_ID"   # all three must be set
```

From a sibling pane (or the herdr UI), confirm herdr sees the OMP pane's live status:

```sh
herdr pane list
herdr wait agent-status <omp-pane-id> --status working --timeout 5000
```

`herdr pane list` (and the sidebar) should show `agent_status` transitioning `idle → working → blocked/idle` as the OMP session runs tools, asks questions, and finishes turns.

herdr also ships its own automated coverage of this integration surface — the asset-level test:

```sh
cd ~/Projects/gyoz-ai/herdr
bun test src/integration/assets/herdr-agent-state.test.ts
```

(this is also what `just test` runs as its `integration-assets-test` step, per the herdr `justfile`.)

# 7. Plugin inventory & portability

Every top-level entry under `~/.omp/agent/extensions/` (confirmed via directory listing — nothing else present):

| Plugin | Purpose | herdr-related? | Portability into the omp fork |
|---|---|---|---|
| `governance/` (`index.ts`, `rules.ts`, `tool-guards.ts`, `bash-policy.ts`) | Enforces the top-priority rules programmatically: injects rule text into every agent's system prompt, blocks/asks on risky bash (git commit/push/reset --hard, publish, sudo, rm -rf /, etc.), blocks Cargo.lock edits, blocks new code comments and test skip-markers, gates the chief's first `task` dispatch behind `memorysearch`, appends a verification checklist to `task` results. | No | **Portable as-is** — pure logic + static text, zero external services, zero machine-specific paths beyond its own `lib/` imports; copy-paste ready as a default extension in the omp fork. |
| `memory/` (`index.ts`, `capture.ts`, `extractor.ts`) | Per-project session memory: boots/reads a local Typesense container, injects `<session_memory>`/`<user_profile>` into the chief's system prompt, tracks `task` dispatch diffs, extracts durable facts on `session_stop` via an LLM call. | No | **Portable with a Docker sidecar** — the code itself is generic, but it hardcodes `TYPESENSE_URL=http://localhost:8108`, a local API key, and paths to `~/.omp/agent/extensions/docker-compose.yml` / a local prompt-count file, and requires a running Docker daemon + Typesense container with accumulated data. Ship it together with `docker-compose.yml` and a documented `docker compose up -d` step, not as a bare drop-in. |
| `ponytail/` (`index.ts`, `doctrine.ts`) | Injects the ponytail minimality-ladder doctrine into every agent's (chief + subagent) system prompt; on `session_stop`, soft-nudges (never hard-blocks) if the last assistant message is missing a `PONYTAIL: PASS` marker. | No | **Portable as-is** — pure static text + string matching, no external deps, no config/env. |
| `herdr-omp-agent-state.ts` | The herdr↔OMP bridge described in §5: reports OMP's live agent state (`working`/`blocked`/`idle`), session identity, and release-on-quit to herdr over a unix socket, gated on `HERDR_ENV`/`HERDR_SOCKET_PATH`/`HERDR_PANE_ID`. | **Yes** | **Not portable as source** — self-declared herdr-installed/managed ("installed by herdr", "reinstalling or updating the integration overwrites this file"); committing it into the omp fork would fight herdr's own installer, which regenerates it on every integration update/reinstall. The omp fork's docs should instead describe the env-var/socket contract (§5) so herdr's installer can keep managing the actual file; any customizations belong in a new sibling extension file, per the header comment. |

`docker-compose.yml` (sibling to the plugins above, not itself a plugin) is the Typesense service definition consumed by `memory/`; it is a portable static asset that should travel with `memory/` under the same Docker-sidecar caveat.
