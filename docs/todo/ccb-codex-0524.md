# CCB Ideas Worth Borrowing For aweteam

**Date:** 2026-05-24

**Reference:** https://github.com/SeemSeam/claude_codex_bridge

**Scope:** Keep aweteam small. Borrow CCB's useful control-plane ideas, not its full daemon/sidebar platform shape.

---

## Product Direction

`aweteam` should remain a thin tmux handoff interface:

- Start one real leader CLI.
- Let the leader create visible worker panes.
- Persist run artifacts under `.aweteam/runs/<run-id>/`.
- Keep the primary UX inside tmux.

The valuable CCB ideas are config authority, task semantics, workspace isolation, lifecycle cleanup, and bounded runtime reads. Avoid copying the full native sidebar, MCP delegation surface, or always-on daemon until aweteam's simpler workflow clearly needs them.

---

## P0 - Highest Value, Small Surface

### 1. Add `aweteam config validate`

**Why:** CCB makes the active config authority explicit. aweteam currently resolves config at run startup, but users cannot inspect what will actually be used before starting a run.

**Todo:**

- [ ] Add `aweteam config validate --config aweteam.json`.
- [ ] Print resolved leader, worker pool, provider, model, `max_instances`, and env reference status.
- [ ] Fail clearly on missing env vars, malformed profile shapes, invalid worker references, and invalid `max_instances`.
- [ ] Add a `--json` output option for automation.

**Acceptance:**

- Running validate does not start tmux or create `.aweteam/runs`.
- Missing `${VAR}` values report the profile name and env key.
- Existing valid example configs pass.

### 2. Add user-level profile inheritance

**Why:** CCB separates built-in, user, and project config authority. aweteam users should not repeat provider endpoints, API env mappings, and model defaults in every project.

**Todo:**

- [ ] Read optional user config from `~/.aweteam/config.json`.
- [ ] Keep project `aweteam.json` as highest authority.
- [ ] Merge `profiles` by profile name, with project values overriding user values.
- [ ] Record both config sources in `config.resolved.json`.
- [ ] Document the merge order in README and README_CN.

**Acceptance:**

- A project can reference a worker profile defined only in `~/.aweteam/config.json`.
- A project can override one field of a user profile without redefining the whole profile.
- Resolved config makes the final authority visible.

### 3. Add task delivery modes: `notify`, `callback`, `silent`

**Why:** CCB distinguishes plain async ask, callback ask, and silent delegation. aweteam has an outbox/inbox protocol already, but every worker currently behaves like the same notification path.

**Todo:**

- [ ] Extend outbox request JSON with optional `mode`.
- [ ] Default to `notify` for backward compatibility.
- [ ] `notify`: current behavior, leader receives creation and completion notices.
- [ ] `callback`: when a worker finishes, inject its result back into the leader pane automatically.
- [ ] `silent`: create and track worker artifacts without leader pane notifications except failures.
- [ ] Persist the mode on each worker entry in `run.json`.

**Acceptance:**

- Existing outbox requests without `mode` still work.
- `callback` sends a bounded result summary plus the `result.md` path, not an unbounded log dump.
- `silent` workers still appear in `aweteam status`.

---

## P1 - Valuable Once P0 Is Stable

### 4. Add opt-in worktree workers

**Why:** CCB's `agent:provider(worktree)` model gives agents isolated working sets. aweteam currently tells workers not to modify source files, which is correct for review tasks but limits implementation workflows.

**Todo:**

- [ ] Add profile field `workspace: "inplace" | "worktree"`.
- [ ] Default to `inplace` with current read-only worker instructions.
- [ ] For `worktree`, create a per-worker git worktree under `.aweteam/worktrees/<run-id>/<worker-name>`.
- [ ] Change worker instructions only for worktree workers: edits are allowed only inside that worktree.
- [ ] On run cleanup, refuse to remove dirty or unmerged worktrees unless forced.

**Acceptance:**

- Inplace workers keep the current "do not modify project files" guardrail.
- Worktree workers start in their own working directory.
- Dirty worktrees are visible in status/cleanup output.

### 5. Add optional compact layout syntax

**Why:** CCB's compact layout is easier to type and explain than full config for simple teams. aweteam should keep JSON as the durable schema, but add compact syntax as an optional convenience.

**Todo:**

- [ ] Support a `layout` field such as `leader:claude; frontend:codex, backend:claude`.
- [ ] Expand layout into leader and worker names during config loading.
- [ ] Reserve `(worktree)` suffix for future/optional worktree mode.
- [ ] Keep explicit `leader`, `workers`, and `profiles` as the canonical resolved shape.

**Acceptance:**

- Layout syntax is optional.
- Explicit JSON still works unchanged.
- Invalid layout errors point to the exact segment that failed.

### 6. Add lifecycle commands: `kill`, `cleanup`, and safe rebuild

**Why:** CCB has a clear start/stop/rebuild lifecycle. aweteam can use a smaller version to prevent stale tmux sessions and stale run directories from accumulating.

**Todo:**

- [ ] Add `aweteam kill <run-id>` to stop the tmux session for one run.
- [ ] Add `aweteam cleanup <run-id>` to archive or remove completed runtime artifacts.
- [ ] Add `aweteam cleanup --stale` for runs whose tmux session no longer exists.
- [ ] Add warnings for running workers and dirty worktrees.

**Acceptance:**

- `kill` does not delete artifacts.
- `cleanup` refuses to remove running sessions by default.
- Status output can distinguish running, stopped, done, failed, and stale runs.

---

## P2 - Defer Until There Is Clear Demand

### 7. Bounded runtime reads for status/watch

**Why:** CCB recently optimized project views with short-lived cache and bounded tail reads. aweteam will need this only when runs/logs get large.

**Todo:**

- [ ] Ensure `aweteam status` reads `run.json` and `status.json` first.
- [ ] Avoid full stdout scans except when a worker is still transitioning.
- [ ] Add bounded tail helpers before adding richer status views.

**Acceptance:**

- `aweteam status --watch` remains responsive with large stdout logs.
- Tests cover large log files without reading them fully in common paths.

### 8. Optional useful-tool / skill projection

**Why:** CCB ships inherited skills and optional useful tools. aweteam may benefit from a small leader helper skill later, but this is not needed for core reliability.

**Todo:**

- [ ] Consider an optional `aweteam-leader` skill that teaches leader agents the outbox protocol.
- [ ] Keep it optional and versioned with the CLI.
- [ ] Do not require global agent skill installation for normal aweteam use.

**Acceptance:**

- aweteam remains usable without installing provider-specific skills.
- Any helper skill mirrors the same protocol documented in generated leader instructions.

---

## Explicit Non-Goals For Now

- [ ] Do not replace JSON config with TOML.
- [ ] Do not build a native sidebar before the tmux UI proves insufficient.
- [ ] Do not add a full always-on daemon as the default runtime.
- [ ] Do not add MCP delegation before the file-based outbox protocol reaches its limits.
- [ ] Do not auto-create worktrees unless a profile explicitly requests it.

---

## Recommended Implementation Order

1. `aweteam config validate`
2. User-level profile inheritance
3. Task delivery modes
4. Lifecycle commands
5. Worktree workers
6. Compact layout syntax
7. Runtime read optimization
8. Optional helper skill projection

