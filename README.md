# SwarmForge — pi-optimized, universal agent support

Fork of [unclebob/swarm-forge](https://github.com/unclebob/swarm-forge) (based on the `four-pack`
workflow) that runs the swarm with **[pi](https://pi.dev)** as the agent harness — while keeping
**universal support** for other agent CLIs, **any model** per role, and **bash** instead of zsh.

The `four-pack` pipeline is unchanged: `specifier` → `coder` → `refactorer` → `architect` with
human approval at the specifier gate.

---

## Quickstart

```bash
# copy this fork's main branch into a project directory
curl -L "https://github.com/pablo-io/swarm-forge/archive/refs/heads/main.tar.gz" | tar -xz --strip-components=1

# edit swarmforge/swarmforge.conf (pick your models, see below), then start
SWARMFORGE_TERMINAL=none ./swarm

# observe/steer agents (headless Linux: attach to sessions manually)
tmux -S "$(cat .swarmforge/tmux-socket)" attach -t swarmforge-coder

# stop the swarm
./close-swarm
```

## Using any model (per role)

Models are **configuration, not code**: the `extra-args` field of `swarmforge.conf` is passed
straight to the agent CLI. To pick a different model, change `--model` (pi) or the equivalent
flag of your backend:

```conf
# pi + models per role (one provider per model family)
window specifier pi master --model opencode-go/deepseek-v4-flash
window coder pi coder --model opencode-go/deepseek-v4-flash
window refactorer pi refactorer batch --model qwen-token-plan/qwen3.7-max
window architect pi architect batch --model qwen-token-plan/glm-5.2

# or a different backend entirely
window coder codex coder --yolo
window architect claude architect batch --dangerously-skip-permissions
```

- List what your harness offers: `pi --list-models` / `pi --list-models <provider>` (run
  `pi update --models` after catalog changes).
- pi supports DeepSeek, Qwen (incl. GLM) and many providers natively — no custom provider
  extensions needed for those.
- Anything after the receive mode (`task`/`batch`) is passed to the agent CLI: `--model`,
  `--thinking`, `--yolo`, etc.

## Frontend design capability

Behavior-driven pipelines verify *what* a UI does, not *how it looks* — without
intervention the agents deliver functional-but-unstyled interfaces. This fork adds
three pieces so projects get real, verifiable UI design:

1. **Design system as raw material** — commit `src/components/ui/` (Card, Input,
   Label, Button, …) + Tailwind theme tokens (`app/globals.css`). Agents compose
   from the system instead of writing raw HTML.
2. **Design as a verifiable rule** — `project.prompt` forbids raw `<input>`/
   `<button>`/`<label>` outside `ui/` and requires the design system in every slice.
3. **A conformance gate** — `tools/ui-conformance <project-dir>` fails the handoff
   if raw HTML leaks, ui/ components are missing, or the system is unused.

The human visual review of the rendered page remains the final aesthetic gate.

## Monitoring tools

The `tools/` folder ships two small bash utilities (no dependencies) for operating a swarm:

- **`tools/sf-queue [project-dir]`** — live handoff queue state for every role: pending (`new`),
  in-process (incl. batches), completed, outbox, sent, plus task names. Reads `.swarmforge/roles.tsv`
  and works with any agent backend.
- **`tools/sf-tokens [project-dir]`** — token consumption per role (input / output / cache / total)
  from pi's session files (`~/.pi/agent/sessions`, organized per worktree). Cumulative across
  restarts. pi-specific (only meaningful when pi is the backend).

```bash
bash tools/sf-queue /path/to/project   # queue state per role
bash tools/sf-tokens /path/to/project  # tokens per role (pi sessions)
```

## Adding another agent backend

The launcher validates backends in `parse-config` and builds the launch command in
`launch-command` (`swarmforge/scripts/swarmforge.bb`). To add a CLI:

1. Add the name to the allowed set in `parse-config`:
   ```clojure
   (when-not (#{"claude" "codex" "copilot" "grok" "pi" "your-agent"} agent)
   ```
2. Add a `case` arm in `launch-command` that starts the CLI interactively with the initial
   prompt, e.g. for pi:
   ```clojure
   "pi" (str "pi -a --name " (sq (str "SwarmForge " display)) " "
             (extra-args-prefix row) "\"$(cat " (sq (str prompt-file)) ")\"")
   ```
3. The binary must be on `PATH` (the launcher checks it). The agent must be an **interactive
   TUI/REPL** that accepts typed input — wake-ups arrive as a typed message + Enter.

Tip: for many backends, a per-harness launch script (like `terminal-adapters/`) is cleaner than
adding `case` arms — the fork is happy to host one per agent.

## Changes & improvements vs upstream

| # | Change | Why |
|---|---|---|
| 1 | **pi as agent backend** | `parse-config` accepts `pi`; launch arm `pi -a --name 'SwarmForge <Role>' [extra] "$(cat prompt)"`. pi starts interactive, sends the initial prompt, stays in the TUI. |
| 2 | **Bash-native, zero zsh dependency** | All shell scripts migrated: shebangs `zsh` → `bash`, `${1:l}` → `tr`, `<->` glob → regex, `&!` → `& disown` (tmux pane shell is bash), `zsh -c` → `bash -c` in `swarmforge.bb`/`swarm-window-watchdog.bb`. Bash is the default on Linux and exists on every platform. |
| 3 | **Universal wake-up (pi + others)** | Upstream typed the wake-up text then a separate Enter; pi-tui drops a standalone CR, so messages sat in the editor. The fork embeds the CR in the same write (`text\r`) and keeps the trailing LF for other TUIs — validated in pi, compatible with claude/codex/grok. |
| 4 | **Self-contained branch** | `swarmforge/scripts/` vendored (upstream ignores it and downloads from `main`); `.gitignore` adjusted; empty `scripts/shared-articles/` stops the wrapper download. No surprise overwrites from upstream. |
| 5 | **Shared constitution articles included** | `engineering.prompt`, `handoffs.prompt`, `workflow.prompt` copied into `constitution/articles/` (upstream `four-pack` only carries `project.prompt`, so agents had no handoff/workflow rules). |
| 6 | **Git identity and worktree failure detection** | Upstream fails *silently* when `git commit` cannot run (no `user.name`/`user.email`) → unborn HEAD → empty worktrees. Now: clear error with instructions before `git init`, and `fail!` if `git worktree add` fails. |

## Scope (verified)

Verified on Linux (Arch) with the real pi CLI and real provider credentials:

- [x] `bb test` — 24 tests, 91 assertions, 0 failures (no zsh required)
- [x] Launch command generation: `pi -a --name 'SwarmForge <Role>' --model <provider/model> "$(cat prompt)"`
- [x] Swarm startup headless (`SWARMFORGE_TERMINAL=none`): tmux sessions, real git worktrees with
      `swarmforge-<role>` branches, agents live with the correct model per role
- [x] Handoff protocol end-to-end: commit → draft → validation → outbox → daemon → inbox
      (generated headers + `merge_and_process` payload) → tmux wake-up → `ready_for_next.sh`
      (task and batch semantics)
- [x] Autonomous chain: coder → refactorer → architect with automatic task pickup and forwarding
- [x] Clean shutdown via `close-swarm`
- [x] Git identity guard: clear failure with instructions when identity is missing
- [x] Universal wake-up: auto-submit in pi with embedded CR; trailing LF retained for other TUIs

Not tested: macOS/Windows terminal adapters (bash-ported from upstream, not exercised on Linux),
window watchdog with a trackable terminal backend (Linux headless mode has no windows by design).

## Requirements

- **bash** (default shell; tmux panes run it). No zsh required.
  - Linux: any modern distro (bash ≥ 4.4). Windows/WSL: fine.
  - macOS: system bash is 3.2 — `close-swarm` uses array append (`sessions+=(...)`, bash ≥ 4);
    use a modern bash (Homebrew) or zsh on macOS.
- `git` with **`user.name` and `user.email` configured** — enforced with a clear error at startup.
- `tmux` ≥ 3.5 (recommended for pi extended keys).
- Babashka (`bb`) — single static binary.
- One or more agent CLIs on `PATH` (pi, codex, claude, …), authenticated for the providers you use.

## Notes

- **This branch is self-contained**: it does not auto-follow upstream `main` script updates
  (scripts are vendored). Rebase/merge manually if you want upstream changes.
- Handoff semantics are unchanged from upstream: only `git_handoff` / `note`, priorities 00–99,
  chain forwarding always, terminal broadcast merge-only. See `swarmforge/handoff-protocol.md`.
- Upstream is zsh-based; if you pull upstream scripts, re-apply the bash migration
  (2 syntax sites + shebangs + `zsh -c` + `&!`).
