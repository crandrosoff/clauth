# Codex parity: the follow-up series (spec, 2026-09-19)

The codex harness landed in #69 (from #45): a second roster in `codex-profiles.toml`, capture and browser login, `clauth start <codex>` under a private `CODEX_HOME`, the `wham/usage` poll, a separate chain with its own walk, and one read-only section on the Overview. Everything else a Claude Code account gets in clauth — actions on its row, a Setup page, a chain editor, the Usage breakdown, the Config keys, the Tokens lens, `delegate` — stops at the harness line, by the parity map in `docs/codex-plan.md`. This spec is the series that carries codex over that line, one slice per PR, each independently mergeable and each keeping the rulings #69 recorded.

Written from a fork, offered for rulings first. Nothing here reverses a #69 decision; where a slice needs one, it says so under "rulings wanted".

```options
Second roster, tab by tab | Keeps #69's two files and every ruling; each tab draws a codex section and acts on codex rows through one selection type; the most mergeable | More code per tab, and the selection type threads through three tabs | chosen
One unified roster | Cleanest UI code, one list everywhere | Reverses #69's core decision, thousands of lines, and the MCP tools would see codex profiles they cannot serve | rejected
Separate codex dashboard | Lowest risk per tab, nothing touches the claude tabs | Two dashboards, and the two chains never sit side by side | rejected
```

```callout decision
decision | The second roster stays, and the tabs learn a second list
One type, `RowSel { Claude(i) | Codex(i) }`, carries the harness with the cursor. Every handler asks "which harness" once and routes to the writer that already exists. Nothing merges the rosters and nothing forks the app.
```

```chips
A follow_active | wip
B overview + setup | planned
C fallback tab | planned
D usage tab | planned
E config tab | planned
F tokens lens | planned
G delegate over codex exec | planned
```

```provenance
source | docs/codex-parity-plan.md, on the crandrosoff/clauth fork, branch docs/codex-parity-plan
builds on | docs/codex-plan.md (#69) and wiki/Codex.md at upstream mommy 37b4ca08
measured | 2026-09-19, codex 0.155.0, a Pro and a Business workspace
```

## Pinned decisions

1. **Two rosters stay.** `profiles.toml` and `codex-profiles.toml` remain disjoint sets in one namespace (`codex-plan.md`, decision 1-2). No slice merges them. The MCP tools keep failing closed on a codex name until slice G gives `delegate` a codex runner.
2. **Tabs learn a second list; they do not learn a second app.** Each tab draws its claude rows as today and a codex section under them (the Overview already does), and the cursor can land on either. One selection type carries the harness with the index, so every action handler asks "which harness" once, at the top, and routes to the writer that already exists for that harness.
3. **Codex writers are the CLI's.** `switch_codex_profile`, the codex login and delete paths, `CodexState::update` — the TUI calls the same functions the CLI does. No second implementation of a codex mutation.
4. **The switch that follows is opt-in.** `follow_active = false` by default, so #69's "a switch moves the marker and nothing else" stays the shipped behavior until an operator turns the key.
5. **No new wire.** The series reads what `wham/usage`, the local `~/.codex/sessions` rollouts and `codex exec` already give. A feature with no OpenAI equivalent (kick, spend ceiling, per-session fallback, keychain) stays out, per the parity map.
6. **Refuse loudly, never silently.** A codex row that cannot take an action says why in the same words the CLI uses (`--with-fallback is not available on a codex profile …`), the way #69 fixed the "profile not found" copy.

## Measured tonight, on codex 0.155.0

- A Pro plan reports **one window**: `token_count.rate_limits.primary.window_minutes = 10080`, `secondary = null` (a live rollout, 2026-09-19). So the empty `5h` cell on a codex row is correct wire data, not a mapping bug. Slice D renders that honestly instead of a dash.
- Every codex turn writes an `event_msg/token_count` event carrying `rate_limits` (used percent, reset epoch, plan) into `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`. That is a passive local feed, fresher than the poll while a session runs. Slice D may read it as a supplement; it is never the only source, because an idle account writes nothing.
- The harness's own tests were verified against codex 0.145; this host runs 0.155 and every `clauth start work` / capture path exercised tonight behaved as the wiki says.

## Architecture: the selection layer

Today the Overview cursor is an index into `config.profiles`, and every key on the Overview, Setup and Fallback tabs reads that index (`tui/app.rs`, the `KeyCode::Char('a')` arm at ~3201 opens `ActionMenuState` on the selected claude profile). Codex rows are drawn from `app.codex_rows` (`tui/app.rs:1573`) after the claude rows (`tui/render/overview.rs:109-122`) and are "never selectable".

The series adds one type and threads it through:

```rust
/// Which row the cursor is on, across both rosters.
pub(crate) enum RowSel {
    Claude(usize),   // index into config.profiles
    Codex(usize),    // index into app.codex_rows
}
```

- The Overview, Setup and Fallback cursors become a `RowSel`. Movement walks claude rows first, then codex rows, honoring the harness filter (`c`): a filter that hides a harness skips its rows.
- Every handler that reads the cursor matches on `RowSel` once. `Claude(i)` takes the existing path unchanged. `Codex(i)` takes the codex branch the slice adds, or a refusal (pinned decision 6).
- `ActionMenuState` gains a `context` variant for a codex row; its `scoped` items are the codex actions (slice B).
- The hot-reload fingerprint already covers `codex-profiles.toml` (#69 folded fix 6); the TUI's `poll_codex_rows` (`tui/app.rs:~10130`) keeps the codex rows fresh on its interval, so a CLI switch shows up in an open TUI.

That is the whole architectural change. Everything else is per-tab work against it.

## The slices

### A. `follow_active`: the operator's codex follows the chain

**Behavior.** With `follow_active = true` in `codex-profiles.toml`, every codex switch — `clauth <codex-name>`, or the chain's auto-switch — also repoints the operator's own login file (`$CODEX_HOME/auth.json`, default `~/.codex/auth.json`) onto the new active profile's store, `~/.clauth/profiles/<name>/auth.json`. A plain `codex` then starts on the chain's pick with no launcher. A running codex adopts the new chain at its next token refresh: it reloads `auth.json` from disk before spending and, when the file changed under it, uses what it finds (`codex-plan.md`, "Verified codex-0.145 behavior"). Off by default (pinned 4).

**Mechanism.** One place, because both switch paths already funnel through it: `actions::switch_codex_profile` (`src/actions.rs:1143`), called by `cmd_switch` (`src/main.rs:~1688`) and by the scheduler's auto-switch (`src/usage/scheduler.rs:~3629`). After `state.set_active(Some(name))` commits:

1. Read the operator slot path: `$CODEX_HOME/auth.json` else `~/.codex/auth.json`.
2. If it is **not** a symlink into `~/.clauth/profiles/*/auth.json`, do nothing. An unadopted login is the operator's own; the key never hijacks it. (An absent file, a regular file, a link elsewhere: all "do nothing".)
3. Else repoint it to `profiles/<name>/auth.json` through the existing tmp-sibling-then-rename helper (`adopt_operator_auth_slot`, `src/actions.rs:1588`, hoisted so both callers share it). The rename is atomic; a codex holding the old file open keeps reading the old chain until its next reload, which is the documented boundary.
4. Log `clauth: <path> now follows '<name>'`. On `SwitchAction::Off` (every member spent, slot cleared) leave the link where it is and log that the operator's codex stays on `'<old>'`: a cleared slot must not strand a working login.

The quarantine guard (`refuse_if_quarantined`) already runs before the marker moves, so the link never lands on a dead chain. On a host without symlinks the operator slot is a separate copy (wiki, "Windows and hosts without symlinks"), which step 2 reads as "not adopted", so the key is inert there and the wiki says so. `clauth delete` of the followed profile keeps its shipped behavior: the link is detached and the operator is told to `codex login` (wiki, "Remove").

**Why this is safe with two chains.** Each profile's chain is one physical file with one refresher; the link only chooses which file the operator's codex reads. No copy of any chain is created (decision 8 in `codex-plan.md`), so the single-use refresh token is never carried twice.

**Tests** (inline, sandbox HOME): knob on + adopted link → switch repoints, and the auto-switch path repoints through the same function; knob off → link untouched; knob on + regular file → untouched; knob on + link to a non-store target → untouched; `Off` → link untouched and the log names the account it stays on.

**Docs.** `wiki/Codex.md` Switch and Auto-switch sections; `wiki/Configuration.md` `codex-profiles.toml` table gains the key.

**Size.** ~60 lines of Rust, ~120 of tests, two wiki paragraphs.

**Rulings wanted.** R1: default off (spec says yes). R2: the key name `follow_active`. R3: on `Off`, leave the link (spec) versus detach it. R3b: on `delete` of the followed profile with the key on, keep the shipped detach (spec) versus repoint to the new active.

### B. Overview and Setup: codex rows take actions

**Behavior.** A codex row can be selected on the Overview. `↵` / `a` opens the action menu with the codex set: `switch` (moves the marker; with slice A on, the link too), `re-login` (browser mint into the same name), `disable` / `enable`, `delete`, `rename`. `⇧↑`/`⇧↓` reorders the codex section in `profiles` display order. The Setup tab shows a codex profile's page with the rows that apply: name, harness, plan, the chain it belongs to, `hooks_json`, the store path; the claude-only rows (endpoint, api key, env, model routing, auto-start) are absent, not greyed.

**Mechanism.** The selection layer above. `ActionMenuState` gets a `Codex` context; the items call `switch_codex_profile`, the codex login path (`src/codex_login.rs`), and the codex delete path #69 added under `CodexState::update`. `disable` / `enable` do not exist for codex today — the CLI refuses them as claude-only (wiki, "When something is refused") — so this slice adds them to `CodexState` first (a `disabled` set the chain walk and the usage poll skip, mirroring the claude semantics) and to the CLI verbs, then the TUI calls those. Reorder writes `profiles` in `codex-profiles.toml`. The Setup page is a second row-set builder beside `config_rows` (`tui/app.rs:6845`) returning only the rows a codex profile has, plus a `+ codex login` row that runs the browser mint with the modal the claude login already has (`r` reopen, `c` copy the link, `p` paste a code) — noting that codex's callback still has to reach this host's loopback port. Every refusal reuses the CLI's wording from the wiki's refusal table.

**Tests.** Cursor movement across the harness boundary under each filter; each action against a sandbox roster; the Setup row-set for a codex profile lists no claude-only row.

**Rulings wanted.** R4: the codex action set above. R5: whether `rename` should be allowed while a live session holds the chain (spec: refuse, same as delete).

### C. Fallback tab: the codex chain editor

**Behavior.** A second section on the Fallback tab, under the claude chain: the codex chain in walk order, with add / remove / reorder, the codex `weekly_switch_threshold` line, `wrap_off`, and (from slice A) `follow_active`. The hand-edited file stops being the only way.

**Mechanism.** The claude editor keys off `cfg.state.fallback_chain` (`tui/render/chain.rs:76,174`, `tui/app.rs:5149 handle_fallback_chain_key`). The codex section reuses the row widgets against `CodexState` and writes through `CodexState::update`. Focus gains a `CodexChain` arm beside `FallbackFocus::Chain`.

**Tests.** Add/remove/reorder round-trips through the file; the threshold clamps to 50–100 as the file rule says; a member removed while active clears the marker the way the CLI does.

**Rulings wanted.** R6: one tab with two sections (spec) versus a `c` filter like the Overview.

### D. Usage tab: codex accounts in the breakdown

**Behavior.** Each codex profile gets its Usage rows: the windows the wire reports, the reset clock, the plan word. A plan with one window shows one bar and says `weekly only`, never a dash that reads as "unknown". Burn and ETA use the same label-driven code (`burn.rs`) since it names no window.

**Mechanism.** The Usage renderer reads per-profile `usage_cache.json` through `profile_cache::load_profile_cache`; codex rows load theirs the same way (`tui/app.rs:1582` already does for the Overview). A `RowSel::Codex` cursor selects a codex row's detail. Optional supplement: the newest `token_count.rate_limits` from `~/.codex/sessions` for a profile with a live session, stamped as `live` in the row the way claude's live column reads.

**Tests.** A one-window cache renders one bar and the `weekly only` label; a two-window cache renders both; the supplement never overrides a fresher poll.

**Docs.** `wiki/Codex.md` says the poll's "two windows" fill the 5h and 7d columns; a Pro or Business plan reports one. The sentence becomes "the windows it reports (one or two)".

**Rulings wanted.** R7: read the rollout `token_count` feed at all (spec: yes, supplement only).

### E. Config tab: the codex keys

**Behavior.** A `codex` group on the Config tab: `weekly_switch_threshold`, `wrap_off`, `follow_active`. Same widgets as the claude keys (`tui/render/global_config.rs`).

**Mechanism.** Rows bound to `CodexState` fields, written through `CodexState::update`. Trivial once A exists; listed as its own slice so C and E can land in either order.

### F. Tokens tab: a codex cost lens

**Behavior.** The Tokens tab totals codex sessions beside Claude Code ones: per day, per model (`turn_context.model`), input / output / cached, and an API-equivalent cost from the price cache, labeled as an estimate exactly like the claude lens.

**Mechanism.** A second feeder beside the claude one (`src/tokens.rs`, `collect_jsonl` at ~963): walk `~/.codex/sessions/**/rollout-*.jsonl`, read `event_msg/token_count` events whose `info` carries the turn's token counts, key by `session_meta.session_id`. `token_ledger` is harness-agnostic already (`codex-plan.md`, settled question 9). The lens gets a harness column and the `c` key the Overview has.

**Tests.** Fixtures from real rollouts (redacted): a session with two turns totals correctly; a rollout with `info: null` events contributes nothing; a session split across two files does not double count.

**Rulings wanted.** R8: the rollout files as the data source (they are codex's own history, not a clauth artifact). R9: pricing: reuse the price cache keyed by the `turn_context.model` string.

### G. `delegate` over `codex exec`, and the Plugin tab

**Behavior.** The MCP `delegate` tool accepts a codex profile and runs the task under `codex exec` in that profile's home; the result comes back in the same shape the claude run returns. The Plugin tab lists codex profiles' live sessions beside the claude ones.

**Mechanism.** `run_delegate` (`src/mcp/mod.rs:3418`) spawns `claude -p … --output-format stream-json`; a codex arm spawns `codex exec` with the profile's `CODEX_HOME` and `-c` overrides that `clauth start` already builds (`src/runtime.rs`), and a per-harness result formatter maps its final message and usage into the delegate result. `AppConfig.profiles` stays claude-only; the MCP layer resolves a codex name through `CodexState` explicitly, so nothing else in the MCP surface changes.

**Tests.** A shimmed `codex exec` produces the documented output; the formatter yields the same result keys as the claude formatter; a codex name that is quarantined is refused with the CLI's words.

**Rulings wanted.** R10: `codex exec`'s output contract to pin (its `--json` event stream versus last-message only). R11: whether `delegate` may run a codex profile that is not the active one (spec: yes, like `clauth start`).

## Delivery

Order A, B, C, D, E, F, G. A ships first and alone, because it is the behavior the series exists for and it touches one function. B carries the selection layer that C and D need. E is a one-evening slice once A's key exists. F and G are independent of each other and of C–E.

Each PR: red tests first, the change, the wiki, the whole suite against a recorded baseline, and a cross-model review before it opens.

## Out of scope, kept from the parity map

Kick / auto-start (no codex endpoint), the spend ceiling and scoped weekly windows (no wire), per-session fallback and `--with-fallback` (codex binds `auth.json` at start), the macOS keychain mirror (forced file mode), `clauth proxy` (separate feature), settings sync and the sessions index (codex owns its own).
