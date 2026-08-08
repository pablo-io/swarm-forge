# SwarmForge — `four-pack-pi-linux`

Fork branch of [unclebob/swarm-forge](https://github.com/unclebob/swarm-forge) (branch `four-pack`)
adapted to run the four-pack workflow with **[pi](https://pi.dev)** as the single agent harness,
**fully on Linux**, **bash-native**, with **one model per role**.

Upstream `four-pack` roles (unchanged semantics): `specifier` → `coder` → `refactorer` → `architect`
with human approval at the specifier gate.

---

## Changes vs upstream

| # | Change | Why |
|---|---|---|
| 1 | **pi as agent backend** (`swarmforge.bb`) | `parse-config` accepts `pi`; launch arm: `pi -a --name 'SwarmForge <Role>' [extra-args] "$(cat prompt)"`. pi starts interactive, sends the initial prompt, and stays in the TUI (wake-ups arrive as typed messages). |
| 2 | **Bash-native, zero zsh dependency** | All 18 shell scripts migrated: shebangs `zsh` → `bash`, `${1:l}` → `tr`, `<->` glob → regex, `&!` → `& disown` (tmux pane shell is bash), `zsh -c` → `bash -c` in `swarmforge.bb` and `swarm-window-watchdog.bb`. |
| 3 | **One model per role** (`swarmforge.conf`) | DeepSeek (opencode-go flash) / Qwen / GLM per role. Model ids verified with `pi --list-models`. No custom pi provider extensions needed. |
| 4 | **Self-contained branch** | `swarmforge/scripts/` vendored (upstream ignores it and downloads from `main`); `.gitignore` adjusted; empty `scripts/shared-articles/` prevents the wrapper download. `bb.edn` + `test/` + `close-swarm` vendored too. |
| 5 | **Shared constitution articles included** | `engineering.prompt`, `handoffs.prompt`, `workflow.prompt` copied into `constitution/articles/` (upstream `four-pack` only carries `project.prompt`, so agents had no handoff/workflow rules). |
| 6 | **Git identity and worktree failure detection** (`swarmforge.bb`) | Upstream fails *silently* when `git commit` cannot run (no `user.name`/`user.email`) → unborn HEAD → empty worktrees. Now: clear error with instructions before `git init`, and `fail!` if `git worktree add` fails. |

## Scope (verified)

Everything below was verified on Linux (Arch) with the real pi CLI (v0.84.1) and real provider credentials:

- [x] `bb test` — 24 tests, 91 assertions, 0 failures (no zsh required)
- [x] Launch command generation: `pi -a --name 'SwarmForge <Role>' --model <provider/model> "$(cat prompt)"`
- [x] Swarm startup headless (`SWARMFORGE_TERMINAL=none`): 4 tmux sessions, 4 real git worktrees with `swarmforge-<role>` branches, 4 pi agents live with the correct model per role
- [x] Handoff protocol end-to-end: commit → draft → `swarm_handoff.sh` validation → outbox → daemon → `inbox/new/` (generated headers + `merge_and_process` payload) → tmux wake-up typed into the pi TUI → `ready_for_next.sh` with batch semantics
- [x] Clean shutdown via `close-swarm`
- [x] Git identity guard: clear failure with instructions when identity is missing

Not tested: macOS/Windows terminal adapters (kept from upstream, bash-ported, not exercised on Linux),
window watchdog with a trackable terminal backend (Linux headless mode has no windows by design),
full four-pack pipeline with human approval (needs a real task and your approval at the specifier gate).

## Prerequisites (Linux)

- `bash` (default shell; tmux panes run it)
- `git` with **`user.name` and `user.email` configured** (global or local) — required for the initial commit
- `tmux` ≥ 3.5 (recommended for pi extended keys)
- Babashka (`bb`) — single static binary
- pi CLI (`@earendil-works/pi-coding-agent`), authenticated with the providers you use
- Provider keys for pi, e.g. `QWEN_TOKEN_PLAN_API_KEY` (used by the default config)

## Quickstart

```bash
# copy this branch into a project directory
BRANCH=four-pack-pi-linux
curl -L "https://github.com/pablo-io/swarm-forge/archive/refs/heads/${BRANCH}.tar.gz" | tar -xz --strip-components=1

# start the swarm (headless on Linux; attach to sessions manually)
SWARMFORGE_TERMINAL=none ./swarm

# observe/steer an agent (from another terminal)
tmux -S "$(cat .swarmforge/tmux-socket)" attach -t swarmforge-coder

# stop the swarm
./close-swarm
```

## Configuration — models per role

`swarmforge/swarmforge.conf`:

```conf
window specifier pi master --model opencode-go/deepseek-v4-flash
window coder pi coder --model opencode-go/deepseek-v4-flash
window refactorer pi refactorer batch --model qwen-token-plan/qwen3.7-max
window architect pi architect batch --model qwen-token-plan/glm-5.2
```

- Any field after the receive mode (`task`/`batch`) is passed to the pi CLI: `--model`, `--thinking`, etc.
- Available models: `pi --list-models` / `pi --list-models <provider>` (run `pi update --models` after catalog changes).
- The `qwen-token-plan` provider exposes DeepSeek, Qwen, **and GLM** models; `opencode-go` also exposes DeepSeek and GLM variants. Both are configured in pi's `auth.json`.

## Notes

- **No zsh required** — this branch is fully bash. Upstream is zsh-based; if you rebase against upstream `main`
  scripts, re-apply the bash migration (2 syntax sites + shebangs + `zsh -c` + `&!`).
- **Git identity is enforced** at startup with a clear error (see Change 6).
- Handoff semantics are unchanged from upstream: only `git_handoff` / `note`, priorities 00–99,
  chain forwarding always, terminal broadcast merge-only. See `swarmforge/handoff-protocol.md`.
