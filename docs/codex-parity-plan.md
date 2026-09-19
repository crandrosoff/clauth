# Codex parity: the follow-up series (spec, 2026-09-19)

The codex harness landed in #69 (from #45): a second roster in `codex-profiles.toml`, capture and browser login, `clauth start <codex>` under a private `CODEX_HOME`, the `wham/usage` poll, a separate chain with its own walk, and one read-only section on the Overview. Everything else a Claude Code account gets in clauth — actions on its row, a Setup page, a chain editor, the Usage breakdown, the Config keys, the Tokens lens, `delegate` — stops at the harness line, by the parity map in `docs/codex-plan.md`. This spec is the series that carries codex over that line, one slice per PR, each independently mergeable. It keeps #69's rulings except for the two amendments named below, which it asks for openly.

Written from a fork, offered for rulings first. Most of #69's rulings are **preserved** unchanged.
Two are **amendments this spec asks for**, and it says so rather than presenting them as settled:
**disable / enable on a codex name**, which settled question 8 refuses (`codex-plan.md:217`), and
**launch-time account selection**, which extends decision 4's "a codex switch writes only
`codex-profiles.toml`" (`codex-plan.md:21`) without changing what a switch writes. Each slice names
its own under "rulings wanted".
<!-- astra-contradictions: the earlier line claimed "nothing here reverses a #69 decision", which
     was inaccurate for B's disable/enable and for the withdrawn A's switch semantics. -->

```options
Second roster, tab by tab | Keeps #69's two files, and every ruling except the two amendments named below; each tab draws a codex section and acts on codex rows through one selection type; the most mergeable | More code per tab, and the selection type threads through three tabs | chosen
One unified roster | Cleanest UI code, one list everywhere | Reverses #69's core decision, thousands of lines, and the MCP tools would see codex profiles they cannot serve | rejected
Separate codex dashboard | Lowest risk per tab, nothing touches the claude tabs | Two dashboards, and the two chains never sit side by side | rejected
```

```callout decision
decision | The second roster stays, and the tabs learn a second list
One type, `RowSel { Claude(i) | Codex(i) }`, carries the harness with the cursor. Every handler asks "which harness" once and routes to the writer that already exists. Nothing merges the rosters and nothing forks the app.
```

```chips
A2 codex shim | planned
B overview + setup | planned
C fallback tab | planned
D usage tab | planned
E config tab | planned
F tokens lens | planned
G delegate over codex exec | planned
```

```provenance
source | docs/codex-parity-plan.md, on the crandrosoff/clauth fork, branch docs/codex-parity-plan
live | https://2026-09-19-clauth-codex-parity-plan.3dstories.ca (doc-harness convention URL, UNVERIFIED from this session: the harness sits behind Cloudflare Access and a made-up subdomain returns the same login page, so a 200 proves nothing)
builds on | docs/codex-plan.md (#69) and wiki/Codex.md at upstream mommy 37b4ca08
measured | 2026-09-19, codex 0.155.0, a Pro and a Business workspace
```

## Pinned decisions

1. **Two rosters stay.** `profiles.toml` and `codex-profiles.toml` remain disjoint sets in one namespace (`codex-plan.md`, decision 1-2). No slice merges them. The MCP tools keep failing closed on a codex name until slice G gives `delegate` a codex runner.
2. **Tabs learn a second list; they do not learn a second app.** Each tab draws its claude rows as today and a codex section under them (the Overview already does), and the cursor can land on either. One selection type carries the harness with the index, so every action handler asks "which harness" once, at the top, and routes to the writer that already exists for that harness.
3. **Codex writers are the CLI's.** `switch_codex_profile`, the codex login and delete paths, `CodexState::update` — the TUI calls the same functions the CLI does. No second implementation of a codex mutation.
4. **A switch still writes only the marker.** #69's decision 4 stands untouched: `clauth <codex-name>` moves the active marker in `codex-profiles.toml` and changes no credential file. What A2 adds is a separate, opt-in **launcher** (`codex_shim = false` by default) that reads that marker at launch. A switch therefore lands at the next `codex` launch, never mid-session — the boundary `clauth start` already states for `--with-fallback` (`src/main.rs:456`).
   <!-- astra-objection-1: the withdrawn slice A made this decision a link mutation and claimed
        live adoption. Both are gone. -->
5. **No new wire.** The series reads what `wham/usage`, the local `~/.codex/sessions` rollouts and `codex exec` already give. A feature with no OpenAI equivalent (kick, spend ceiling, per-session fallback, keychain) stays out, per the parity map.
6. **Refuse loudly, never silently.** A codex row that cannot take an action says why in the same words the CLI uses (`--with-fallback is not available on a codex profile …`), the way #69 fixed the "profile not found" copy.

## Measured tonight, on codex 0.155.0

- **One observed rollout**, on one Pro account, 2026-09-19, reported a single window: `token_count.rate_limits.primary.window_minutes = 10080`, `secondary = null`. That is a 7-day window **by its duration**, which is how the existing contract maps windows (`codex-plan.md:44,205,212`) — not by position and not by plan name. The sample supports one claim only: *this* account's *rollout* feed reported one window on that day. It does **not** establish that every Pro or Business account reports one window, and it does **not** by itself prove the empty `5h` cell is correct, because the cell is fed by the `wham/usage` **poll**, which this sample did not capture. Slice D must obtain a matching poll fixture before the `5h` dash is called correct data.
  <!-- astra-objection-7: the earlier bullet generalized one rollout to "a Pro plan reports one
       window", conflated the rollout feed with the poll, and used "weekly" as a plan property
       rather than a duration. -->
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
- The hot-reload fingerprint already covers `codex-profiles.toml` (#69 folded fix 6); the TUI's `poll_codex_rows` (`src/tui/app.rs:10127-10137`) keeps the codex rows fresh on its interval, so a CLI switch shows up in an open TUI.
- **A bare index is not an identity, and this is a correctness rule, not a nicety.** `poll_codex_rows` replaces `app.codex_rows` **wholesale every second**, ungated by tab and by filter (`src/tui/app.rs:10127-10137`). A `clauth delete` or a reorder in another terminal can therefore make `Codex(1)` name a **different account** between the moment the operator selects it and the moment the action runs. So:
  - **The cursor itself survives a reload by identity, not by index.** `poll_codex_rows` replaces the vector every second whether or not a menu is open (`src/tui/app.rs:10135-10136`), so an index retained across that replacement can already name another account **before** any menu opens. On each replacement the cursor is re-resolved to the profile name it held; when that name is gone, the cursor moves to a defined neighbor rather than staying on a stale index.
  - `RowSel` is resolved to a **profile name** the instant a menu, modal or confirmation opens, and that name — with its harness — is frozen in the action state.
  - Before any mutation, the frozen target is **revalidated** against the current roster. A target that no longer exists, or whose row moved, refuses with the roster's own words and closes the menu.
  - `RowSel` represents an **empty selection** explicitly (an empty roster, or a filter that hides every row), rather than defaulting to index 0.
  - Tests: delete and reorder a codex profile while its action menu is open, and assert the mutation lands on the frozen name or refuses — never on whatever now sits at that index.
  <!-- astra-objection-5: the earlier draft carried only `RowSel::Codex(usize)` plus a hot-reload
       paragraph, which prevents a cross-harness mistake but not a wrong-account mutation. -->

That is the whole architectural change. Everything else is per-tab work against it.

## The slices

### A2. The `codex` shim: launch-time account selection

<!-- astra-objection-1: slice A (live adoption by repointing the operator's auth.json link)
     is WITHDRAWN. codex 0.155's `reload_if_account_id_matches` makes a running codex REFUSE a
     reloaded auth.json naming a different account, and codex persists refreshed tokens BY PATH,
     so a repoint during an in-flight refresh can write account A's tokens into B's store.
     A2 replaces it: choose the account at launch, mutate no link, claim no live adoption. -->

**What it is, in one sentence.** A small generated `codex` program early on your `PATH` that starts
the codex chain's active profile through `clauth start`, so typing `codex` runs the account clauth
says is active, in that account's own home, with nothing repointed and nothing adopted.

**Why not the link.** The withdrawn slice A repointed `$CODEX_HOME/auth.json` and claimed a running
codex would adopt the new account at its next refresh. It will not. codex 0.155's auth manager
compares the reloaded account against the cached one in `reload_if_account_id_matches`, and
`refresh_token` returns a permanent account-mismatch error rather than adopting the new account
([codex 0.155 `login/src/auth/manager.rs`](https://raw.githubusercontent.com/openai/codex/rust-v0.155.0/codex-rs/login/src/auth/manager.rs)).
Worse, codex persists refreshed tokens by PATH: repointing the link between a refresh request and
its response can merge account A's replacement tokens into account B's store and leave A holding a
spent token ([codex 0.145 `manager.rs`](https://raw.githubusercontent.com/openai/codex/rust-v0.145.0/codex-rs/login/src/auth/manager.rs),
[`storage.rs`](https://raw.githubusercontent.com/openai/codex/rust-v0.145.0/codex-rs/login/src/auth/storage.rs)).
An atomic rename of the *symlink* does not make the *read → network refresh → persist* transaction
atomic. So A2 never writes a credential file at all.

**The boundary, stated plainly.** A switch lands at the **next `codex` launch**, never mid-session.
Said precisely: **which account a running codex spends is fixed at start; that same account's
credentials can still refresh while it runs.** `clauth start` already refuses `--with-fallback` on
exactly this boundary (`src/main.rs:453-458`), though its wording — *"codex reads auth.json once at
start"* — is loose: codex does reload the file, and then checks that the account matches
(`reload_if_account_id_matches`). A2 does not narrow the boundary and does not claim to.
<!-- verify-A5: "reads auth.json once" is not literally true. The reload happens; the ACCOUNT is
     what is pinned. Stating it loosely here would repeat the mistake that sank slice A. -->

**Behavior.**

- `codex_shim = false` in `codex-profiles.toml`, default off. Turning it on makes clauth generate
  the shim. Turning it off removes it. The key is the intent, the file is derived from it, and one
  converge function makes the file match the key (see the failure contract below).
- With the shim on `PATH` ahead of the real codex, a **session-starting** `codex <args>` runs
  `clauth start <active-codex-profile> -- <args>`. The session gets that profile's own
  `CODEX_HOME` (`~/.clauth/profiles/<name>/codex-home-<sid>`), which is exactly what
  `clauth start <name>` gives today (`wiki/Codex.md`, "Run"). Args after `--` reach codex verbatim,
  which is the documented `clauth start` contract already.
- **The shim routes an ALLOWLIST, and everything else execs the real codex unchanged.** This is the
  single most important rule in A2, and it is an allowlist rather than a denylist on purpose: a
  codex subcommand added in a future release then defaults to the safe side instead of silently
  acquiring a managed account.
  - **Routed through clauth:** no subcommand at all (the interactive session), `exec`, `resume`,
    `fork`, `review`. These start or continue a session, which is what the shim exists for.
  - **Passed straight to the real codex, with the operator's own environment untouched:**
    everything else, and **`login` and `logout` above all**.

  **Why that rule exists, and what it prevents.** `clauth start` pins `CODEX_HOME` to the profile's
  runtime home and forces `cli_auth_credentials_store="file"`, then appends the caller's arguments
  verbatim (`codex_spawn_command`, `src/start.rs:559-573`); that home's `auth.json` is a link onto
  the profile's own credential store (`src/runtime.rs:5401-5405`). So a naive shim would turn a
  plain `codex login` into a login **against the active managed profile**, truncating and replacing
  that profile's chain — and `wiki/Codex.md` already warns in its own words that codex's login and
  logout *"revoke the login they find"*, server-side, with no re-capture able to bring it back.
  A2 must not re-create by accident the exact hazard the wiki warns about.
  <!-- verify-A6 (CONFIRMED, the severe one): the reviewer showed that forwarding every argument
       makes `codex login` write into the active profile's store. Confirmed against
       `src/start.rs:559-573` and `src/runtime.rs:5401-5405`. The allowlist is the repair. -->

  **What A2 does NOT fix here.** If the operator's own `~/.codex/auth.json` is already an adopted
  link from `clauth login <name> --codex`, then a passed-through `codex login` still reaches the
  profile's chain — that is the pre-existing hazard the wiki documents, and it behaves exactly as
  it does with no shim installed. A2 neither worsens it nor repairs it.
- **No active codex profile** (the marker is unset, or `SwitchAction::Off` cleared it): the shim
  execs the **real codex** with the arguments unchanged. Typing `codex` must never fail because
  clauth has nothing to offer. Nothing is printed on this path.
- **Inside a clauth codex session** the shim steps aside and execs the real codex, so a `codex`
  typed inside `clauth start work` stays in `work`'s home rather than starting a second session.
  This mirrors the cross-harness scrub clauth already does for a claude session started inside a
  codex one (`src/harness.rs:107-116`).
  <!-- astra-objection-2: slice A could not tell an operator slot from a managed session slot, so
       a switch would repoint a RUNNING session's private link while the registry still named the
       old profile. A2 has no such ambiguity: it mutates nothing, and a managed session is
       recognized by the bypass env key clauth itself sets on the spawn. The ownership-check
       weakness Astra found in `clauth_auth_store_owner` (`src/actions.rs:1573`, which matches the
       `profiles/<name>/auth.json` SUFFIX rather than membership under the real clauth root) is
       untouched by A2, because A2 reads no link — it stays an open upstream question, recorded
       here so it is not lost with slice A. -->
- **Windows: no shim, and clauth says so.** `clauth codex shim install` refuses on Windows with a
  reason, rather than installing something that half works. `resolve_cli_command` enumerates every
  `PATHEXT` match in `PATH` order and **prefers a native `.exe` before falling back to the first
  match** (`src/runtime.rs:3575-3583`). The divergence is therefore **configuration-dependent, not
  automatic**: with a `codex.cmd` shim in an EARLIER `PATH` directory and a real `codex.exe` in a
  LATER one, a plain shell runs the shim while clauth skips it and runs the real binary — two
  different `codex` programs depending on who asks. That configuration is the normal one for a shim,
  which is why refusing is the honest answer. Not executed on Windows; read from source only.
  <!-- verify-A3: the earlier wording said "beside each other", which implied the divergence is
       inevitable whenever both exist. It is not; it depends on PATH order. -->

**Mechanism.**

1. **A new start target, so the shim holds no logic.** `clauth start --codex-active -- <args>`
   resolves `CodexState::active_profile()` and starts it, or — when there is no active profile —
   execs the real codex with `<args>` unchanged. Every decision stays in Rust, where it is testable.
   The shim never parses TOML and never reads the roster.
2. **The shim file**, generated at `<data_dir>/clauth/bin/codex` (`~/.local/share/clauth/bin/codex`
   on Linux, `~/Library/Application Support/clauth/bin/codex` on macOS). clauth already owns
   `<data_dir>/clauth` — that is where the Claude Code plugin materializes
   (`src/plugin_host.rs:442-447`) — so this adds a sibling `bin/`, not a new root.
   **Why not `<data_dir>/clauth/current@claude/bin`, which does appear on some `PATH`s.** That
   directory is agentgear's **plugin** materialization tree (`expected_pointer`,
   `src/plugin_host.rs:441-447`); on this host it holds `.claude-plugin/` and `hooks/` and **has no
   `bin/` at all**. A `.../current@claude/bin` entry *is* present on the `PATH` of a Claude Code
   session — Claude Code adds each installed plugin's `bin/`, and clauth is registered at that path
   in `~/.claude/settings.json` — but it is **absent from a clean login shell** (both checked on
   this host, 2026-09-19). A shim placed there would work inside Claude Code and nowhere else,
   which is the opposite of what an operator-facing `codex` needs. Hence a separate `bin/`.
   <!-- correction-shim-dir + verify-A2: the first draft said that directory is "on no PATH",
        which is too broad — it IS on a Claude Code session's PATH, just not the operator's. The
        design conclusion is unchanged and now rests on the right reason. -->

   ```sh
   #!/bin/sh
   # Generated by clauth. Do not edit — `clauth codex shim install` rewrites it.
   if [ -n "${CLAUTH_CODEX_SHIM_BYPASS-}" ]; then
     exec "@REAL_CODEX@" "$@"
   fi
   exec "@CLAUTH_BIN@" start --codex-active -- "$@"
   ```

   The snippet above is **illustrative**: it shows the routing decision, not the allowlist or the
   stale-path fallback, both of which the generated file must also implement.

   `@REAL_CODEX@` and `@CLAUTH_BIN@` are **absolute paths baked in at generation time**. Resolution
   must reject a candidate by **canonical identity**, not by directory: excluding only the shim's
   own directory is not enough, because a symlink in another `PATH` directory can point back at the
   shim, and baking that alias as `@REAL_CODEX@` makes the bypass branch exec itself forever.
   Generation **and** the stale-path fallback both resolve every candidate to its real path and
   reject any that is the shim.
   <!-- verify-A1b: the reviewer's alias counterexample. A directory exclusion does not survive a
        symlink alias; canonical-identity rejection does. -->
3. **The recursion guard. The env key is load-bearing; the baked path is not a substitute.**
   clauth spawns codex through `codex_command()`, a bare `PATH` lookup on unix
   (`src/runtime.rs:3560-3586`), and would otherwise find the shim. So `CodexEngine` sets
   `CLAUTH_CODEX_SHIM_BYPASS=1` on every codex spawn, beside the keys it already scrubs
   (`CODEX_MANAGED_ENV_KEYS`, `src/harness.rs:149-159`). **That key alone is what breaks the
   loop.** An earlier draft of this spec claimed the baked absolute path was independently
   sufficient. It is not: baking a path into the shim does not change where *clauth* looks, so
   without the key the cycle `clauth → PATH codex → shim → clauth` still closes. The baked path is
   defense in depth against a `PATH` that no longer contains the real codex, nothing more.
   <!-- verify-A1: the "either alone is sufficient" claim was false and is corrected here rather
        than softened. -->
4. **PATH is the operator's to edit.** clauth prints the exact line to add
   (`export PATH="$HOME/.local/share/clauth/bin:$PATH"`) and never writes a shell rc file. `install`
   reports whether the directory is already on `PATH` and whether anything else on `PATH` named
   `codex` comes first.
5. **A stale bake.** If `@REAL_CODEX@` no longer exists (an npm reinstall moved it), the shim falls
   back to a `PATH` walk that skips its own directory, runs what it finds, and prints one line on
   stderr naming `clauth codex shim install` as the repair. A moved codex must not make `codex`
   stop working.

**Failure and consistency contract.**
<!-- astra-objection-3: slice A had no state/link consistency or partial-failure contract.
     A2's equivalent is stated here rather than assumed. -->

- **Converge is serialized against the state, and re-reads the intent it is about to enforce.**
  Committing the key and converging afterwards is not enough on its own: `install` can commit
  `true`, `uninstall` can then commit `false` and remove the file, and `install`'s delayed converge
  can recreate a shim the roster says should not exist (`CodexState::update` saves and releases,
  `src/codex_profiles.rs:191-199`). So converge runs **under the same state lock**, re-reads the
  current key, and enforces *that* value — never the value its caller happened to write. Every
  entry point converges through the one function: the CLI, the TUI Config toggle, and `doctor`.
  <!-- verify-P3: the reviewer's competing-writer counterexample. Re-reading the intent under the
       lock is what makes the last writer win rather than the last converger. -->
- One converge function owns the file: given the key, it writes, rewrites or removes
  `<data_dir>/clauth/bin/codex` and returns what it did. `install` / `uninstall` set the key inside
  `CodexState::update`'s lock, so a failed save never
  leaves a shim the roster does not claim.
- A converge failure is reported with the reason and the key's value, never announced as success.
  The withdrawn slice A logged success unconditionally while its helper returned `false`.
- Converge is idempotent and runs on `install`, `uninstall`, and `clauth doctor`. It does **not**
  run on every switch: a switch writes no file under this design, which is the point.
- `SwitchAction::Off` needs no special case. The shim asks for the active profile at launch, finds
  none, and execs the real codex. Compare slice A, which had to reason about a cleared marker
  stranding a link (`src/usage/scheduler.rs:3637` bypasses `switch_codex_profile` entirely).

**What A2 does NOT claim.** It does not make a running codex change accounts. It does not
coordinate two writers on one credential file — clauth's stand-down still depends on
`has_live_session` (`src/codex_auth.rs:894`), which reads clauth session markers
(`src/runtime.rs:635`), and a bare operator codex writes no such marker. A2 neither fixes nor
worsens that; it simply stops adding a new writer.

**Tests** (inline, sandbox HOME, a fake `codex` on a sandbox `PATH`):

- Key on → converge writes an executable shim whose baked paths are absolute and whose
  `@REAL_CODEX@` is not the shim itself; key off → converge removes it; both are idempotent.
- **`login` and `logout` are NOT routed through clauth.** Stage a fake codex, run the shim with
  `login`, and assert the fake receives `login` with the operator's own environment — no
  `CODEX_HOME` pinned to a profile, no `clauth start` in the chain. The same for `logout`. This is
  the test that would have caught the hazard the reviewer found, so it is written first.
- A subcommand the allowlist does not name (stand in an invented `codex frobnicate`) execs the real
  codex unchanged, proving the default is pass-through and not route-through.
- Each allowlisted form (`<no subcommand>`, `exec`, `resume`, `fork`, `review`) DOES route through
  `clauth start`.
- **The symlink alias case.** Put a symlink to the shim in a second `PATH` directory, generate, and
  assert the baked `@REAL_CODEX@` resolves to neither the shim nor its alias. Run the stale-path
  fallback with the same layout and assert it makes the same rejection.
- **The converge race.** Commit `true`, then commit `false` and remove the file, then run the first
  caller's delayed converge, and assert no shim exists — converge enforced the current key, not its
  caller's.
- `start --codex-active` with an active profile starts that profile, under its own `CODEX_HOME`.
- `start --codex-active` with **no** active profile execs the real codex with the arguments
  unchanged and prints nothing.
- Args after `--` reach codex verbatim, including a flag both programs spell.
- A codex spawned by `clauth start` carries `CLAUTH_CODEX_SHIM_BYPASS`, so the shim on `PATH` is
  not re-entered (assert on the built `Command`'s env, the way the scrub tests do).
- The shim with the bypass set execs the real codex and never calls clauth.
- A baked `@REAL_CODEX@` that no longer exists falls back to the `PATH` walk, skips the shim's own
  directory, and names the repair on stderr.
- Windows: `codex shim install` refuses, with the `.exe`-over-`.cmd` reason in the message.
- A state-save failure inside `install` leaves no shim on disk.

**Docs.** `wiki/Codex.md` gains a "Type `codex` and get the active account" section carrying the
next-launch boundary in the same words as the `--with-fallback` refusal.
`wiki/Configuration.md`'s `codex-profiles.toml` table gains `codex_shim`.
`wiki/Install.md` gains the `PATH` line.

**Size.** ~120 lines of Rust (the converge function, the start target, the env key), ~200 of tests,
three wiki sections. Larger than the withdrawn A, and it ships a behavior that holds.

**Rulings wanted.** R1: the key name `codex_shim`, default off. R2: the shim location
`<data_dir>/clauth/bin/codex`, with `PATH` left to the operator. R3: with no active codex profile,
fall through to the real codex (spec) versus refuse with a message. R3b: spell the target as
`clauth start --codex-active` (spec) versus a reserved name. R3c: Windows — refuse (spec) versus
ship a `.cmd` shim and change `resolve_cli_command`'s `.exe` preference. **R3d, and this is the one
I would most like you to look at:** the routed allowlist (`<no subcommand>`, `exec`, `resume`,
`fork`, `review`) with everything else — `login` and `logout` above all — passed straight to the
real codex. An allowlist is the spec's answer because a future codex subcommand then defaults to
safe. Name anything you would add or remove.

### B. Overview and Setup: codex rows take actions

**Behavior.** A codex row can be selected on the Overview. `↵` / `a` opens the action menu with the codex set: `switch` (moves the marker, and nothing else — pinned decision 4), `re-login` (browser mint into the same name), `disable` / `enable`, `delete`, `rename`. `⇧↑`/`⇧↓` reorders the codex section in `profiles` display order. The Setup tab shows a codex profile's page with the rows that apply: name, harness, plan, the chain it belongs to, `hooks_json`, the store path; the claude-only rows (endpoint, api key, env, model routing, auto-start) are absent, not greyed.

**Mechanism.** The selection layer above. `ActionMenuState` gets a `Codex` context; the items call `switch_codex_profile`, the codex login path (`src/codex_login.rs`), and the codex delete path #69 added under `CodexState::update`. Reorder writes `profiles` in `codex-profiles.toml`. The Setup page is a second row-set builder beside `config_rows` (`tui/app.rs:6845`) returning only the rows a codex profile has, plus a `+ codex login` row that runs the browser mint with the modal the claude login already has (`r` reopen, `c` copy the link, `p` paste a code) — noting that codex's callback still has to reach this host's loopback port. Every refusal reuses the CLI's wording from the wiki's refusal table.

**Two of these actions do not exist for codex yet, and this slice must define them before it calls them.**
<!-- astra-objection-4: the earlier draft routed rename and disable to "the writer that already
     exists". For codex, neither writer exists, and disable's claude semantics are more than
     "skip usage and the chain". Both are specified here instead. -->

- **`rename`.** There is no codex rename writer to route to. `rename_profile` takes an `AppConfig` and moves Claude Code state (`src/actions.rs:991-996`). A codex rename must define all of: the **roster** entry in `profiles`, the **chain** entry in `fallback_chain`, the **active marker** when it names the old name, the **profile directory** `~/.clauth/profiles/<name>/` (which holds `auth.json`, `codex-home/` and the usage cache), the **rotation guard** the claude rename already takes, and the **operator `auth.json` link** when one points into the old directory — capture installs exactly such a link (`wiki/Codex.md`, "Adopt the login your own codex holds"), and moving the directory without repointing or detaching it strands the operator's own codex on a path that no longer exists. Recovery is specified too: a rename that fails after the directory move must leave the roster and the directory naming the same profile, or refuse before moving anything.
- **`disable` / `enable`.** Settled question 8 refuses these on a codex name (`codex-plan.md:217`), so this is an **amendment**, named as one in the header. Claude's `disable_profile` is not merely "skip the chain and the poll": it refuses the **active** profile (`'<name>' is the active account, switch away first`) and a profile holding a **live session** (`'<name>' has a live session, close it first`) — `src/actions.rs:1638-1651`. The codex twin must carry both refusals in the same words, and every codex entry point that can land on a profile (`switch`, `start`, the chain walk, the usage refresh, and later `delegate`) must reject a disabled target rather than silently using it.
- If upstream would rather not take those invariants in this slice, **B narrows to `switch`, `re-login`, `delete` and reorder**, and rename plus disable/enable move to their own PR. That is the fallback this spec is happy with.

**Tests.** Cursor movement across the harness boundary under each filter; each action against a sandbox roster; the Setup row-set for a codex profile lists no claude-only row; rename moves roster, chain, marker and directory together, and refuses when the operator link would be stranded; disable refuses the active profile and a live one in the claude wording; every codex entry point rejects a disabled target.

**Rulings wanted.** R4: the codex action set above. R4b: whether rename and disable/enable belong in B at all, or in their own PR (spec: happy either way). R5: whether `rename` should be allowed while a live session holds the chain (spec: refuse, same as delete).

### C. Fallback tab: the codex chain editor

**Behavior.** A second section on the Fallback tab, under the claude chain: the codex chain in walk order, with add / remove / reorder, the codex `weekly_switch_threshold` line, `wrap_off`, and (from A2) `codex_shim`. The hand-edited file stops being the only way.

**Mechanism.** The claude editor keys off `cfg.state.fallback_chain` (`tui/render/chain.rs:76,174`, `tui/app.rs:5149 handle_fallback_chain_key`). The codex section reuses the row widgets against `CodexState` and writes through `CodexState::update`. Focus gains a `CodexChain` arm beside `FallbackFocus::Chain`.

**Removing a member from the chain is not deleting a profile, and must not behave like one.**
<!-- astra-objection-8: the earlier test asserted that removing an active member clears the
     marker "the way the CLI does". No such CLI precedent exists for chain editing. -->
`CodexState::remove_profile` clears the active marker (`src/codex_profiles.rs:170-175`), but that is the **delete-a-profile** writer, not a chain editor. The claude chain editor removes a member and leaves the marker alone (`remove_chain_member`, `src/tui/app.rs:6292-6304`), and the codex walk simply declines to run when the active profile sits outside the chain — `snapshot_codex_chain` returns `None` (`src/fallback.rs:989-993`). So the codex chain editor **preserves the marker** and matches the claude editor. If upstream wants the other behavior, that is a new ruling (R6b), not existing semantics.

**Tests.** Add/remove/reorder round-trips through the file; removing the **active** member leaves the marker set and the walk declines, matching `remove_chain_member` and `snapshot_codex_chain`; an out-of-band `weekly_switch_threshold` (a hand-edited `0.98`, a `nan`, a `20`) **resets to the 98.0 default** rather than clamping into 50-100 — `weekly_switch_threshold_pct` filters then falls back to `DEFAULT_WEEKLY_SWITCH_PCT` (`src/codex_profiles.rs:132-136`, `src/profile.rs:1030`), and the editor must show the same number the walk will use.
<!-- astra-copy-fix-C: the earlier text said "clamps to 50-100". -->

**Rulings wanted.** R6: one tab with two sections (spec) versus a `c` filter like the Overview. R6b: chain removal preserves the active marker (spec, matching the claude editor) versus clearing it.

### D. Usage tab: codex accounts in the breakdown

**Behavior.** Each codex profile gets its Usage rows: the windows the wire reports, the reset clock, the plan word. Burn and ETA use the same label-driven code (`burn.rs`) since it names no window.

**Windows are labelled from their own duration, never from a plan name or a position.** That is the existing contract (`codex-plan.md:44,205,212`) and this slice keeps it: a `window_minutes` of 10080 renders as a 7-day window, 300 as a 5-hour one, and an unrecognized duration renders with its own duration spelled out rather than being forced into a named column. A response carrying **one** window shows one bar. Crucially, the UI distinguishes three different states that a dash today collapses into one: **a window the wire did not report**, **a poll that has not run yet**, and **a poll that failed**. Only the first is "this account has one window".
<!-- astra-objection-7: the earlier draft said "a plan with one window says `weekly only`", which
     treated a single observation as a plan property and used a name where the contract uses a
     duration. -->

**Mechanism.** The Usage renderer reads per-profile `usage_cache.json` through `profile_cache::load_profile_cache`; codex rows load theirs the same way (`tui/app.rs:1582` already does for the Overview). A `RowSel::Codex` cursor selects a codex row's detail, resolved to a profile name at open time (see the selection layer). Optional supplement: the newest `token_count.rate_limits` from the rollout roots below, for a profile with a live session, stamped as `live` in the row the way claude's live column reads.

**The rollout roots, enumerated — `~/.codex/sessions` alone is the wrong set.**
<!-- astra-objection-6: the earlier draft scanned only `~/.codex/sessions`, which misses every
     session this integration itself launches, and it inferred account ownership from the
     active marker. Both are corrected here. -->

1. `~/.clauth/profiles/<name>/codex-home/sessions/` — the durable per-profile store. **Every shared `clauth start <name>` session writes here**, because the session home links `sessions/` into the store (`wiki/Codex.md`, "Run"). This is the primary root, and the one the earlier draft missed.
2. `~/.clauth/profiles/<name>/codex-home/archived_sessions/` — archiving a thread keeps it; it must not vanish from the totals.
3. The **operator home**: `$CODEX_HOME/sessions` **and `$CODEX_HOME/archived_sessions`** when that variable is set, else the same two under `~/.codex`. Both, because clauth already treats them as one pair — `CODEX_ROLLOUT_ROOTS = ["sessions", "archived_sessions"]` (`src/runtime.rs:5238-5247`), and archiving MOVES a rollout between them. It is one root pair, not necessarily one account.
   <!-- verify-P6: the earlier draft listed archives for the profile store but not for the operator home, and F inherited the omission. -->
4. `--isolated` sessions write into a per-session home that is **removed at exit** (`wiki/Codex.md`, "Run"). Their rollouts are unrecoverable by design. This is a stated coverage gap, not a bug to fix, and the UI must not imply the totals are complete.

**Deduplication.** A shared session's rollout is reachable by two paths — through the session home's link and through the store itself. Walk each root, resolve every candidate to its canonical path, and key by that. A file counted twice is a wrong number presented confidently.

**Account attribution, and when to refuse it.** A rollout under `profiles/<name>/codex-home/` is attributable to `<name>` **by path**, which is the only strong attribution available. A rollout under the operator home is **not** attributable: that directory can hold several accounts' history from before clauth, or from a `codex login` the operator ran themselves. Ownership is therefore **never inferred from today's active marker or from where the `auth.json` link currently points** — that is a statement about now, not about the day the rollout was written. An unattributable rollout is reported under an explicit `unattributed` bucket, never folded into a named profile's number.

**Tests.** A one-window cache renders one bar labelled by its duration; a two-window cache renders both; an unreported window, an un-run poll and a failed poll each render distinguishably; the supplement never overrides a fresher poll; a rollout reachable through both a session home link and the store is counted once; a rollout in the operator home lands in `unattributed` and never in a named profile.

**Docs.** `wiki/Codex.md` says the poll's "two windows" fill the 5h and 7d columns. The sentence becomes "the windows it reports, labelled by their duration", with no claim about which plans report how many — this spec has one rollout sample and no matching poll fixture.

**Rulings wanted.** R7: read the rollout `token_count` feed at all (spec: yes, supplement only). R7b: the `unattributed` bucket for operator-home rollouts (spec) versus excluding them entirely.

### E. Config tab: the codex keys

**Behavior.** A `codex` group on the Config tab: `weekly_switch_threshold`, `wrap_off`, `codex_shim`. Same widgets as the claude keys (`tui/render/global_config.rs`).

**Mechanism.** Rows bound to `CodexState` fields, written through `CodexState::update`. `codex_shim` is the one key with a side effect: toggling it runs A2's converge function **after** the state save commits, and reports what converge did — or why it failed — rather than assuming it worked. Trivial once A2 exists; listed as its own slice so C and E can land in either order.

**Tests.** Toggling `codex_shim` on writes the shim and off removes it; a converge failure surfaces as an error with the key's value, not as a silent success.

### F. Tokens tab: a codex cost lens

**Behavior.** The Tokens tab totals codex sessions beside Claude Code ones: per day, per model (`turn_context.model`), input / output / cached, and an API-equivalent cost from the price cache, labeled as an estimate exactly like the claude lens.

**Mechanism.** A second feeder beside the claude one (`src/tokens.rs`, `collect_jsonl` at ~963): walk `rollout-*.jsonl` under **every root slice D enumerates** — the per-profile store, its `archived_sessions`, and the operator home — read `event_msg/token_count` events whose `info` carries the turn's token counts, and key by `session_meta.session_id`. `token_ledger` is harness-agnostic already (`codex-plan.md`, settled question 9).

F inherits D's root list, its canonical-path deduplication, its `unattributed` bucket for operator-home rollouts, and its stated `--isolated` coverage gap **without restating them** — one enumeration, two consumers.
<!-- astra-objection-6: the earlier draft walked `~/.codex/sessions` only, which misses every
     rollout a `clauth start` session writes. -->

The lens gets a harness column. **The `c` key is already taken on this tab** — `handle_tokens_key` binds `c` to the persisted cache-counting toggle on both views (`src/tui/app.rs:3403-3411`) — so the harness filter takes `h`, or `c` is explicitly reassigned and the cache toggle moved. The spec takes `h`.
<!-- astra-copy-fix-F: the earlier draft said "the `c` key the Overview has", which would have
     collided with the existing cache toggle. -->

**Tests.** Fixtures from real rollouts (redacted): a session with two turns totals correctly; a rollout with `info: null` events contributes nothing; a session split across two files does not double count; a rollout reachable through both a session-home link and the profile store is counted once; an operator-home rollout lands in `unattributed`; `h` filters by harness and `c` still toggles cache counting.

**Rulings wanted.** R8: the rollout files as the data source (they are codex's own history, not a clauth artifact). R9: pricing: reuse the price cache keyed by the `turn_context.model` string. R9b: `h` for the harness filter (spec) versus reassigning `c` and moving the cache toggle.

### G. `delegate` over `codex exec`, and the Plugin tab

**Behavior.** The MCP `delegate` tool accepts a codex profile and runs the task under `codex exec` in that profile's home; the result comes back in the same shape the claude run returns. The Plugin tab lists codex profiles' live sessions beside the claude ones.

**Mechanism.** `run_delegate` (`src/mcp/mod.rs:3418`) spawns `claude -p … --output-format stream-json`; a codex arm spawns `codex exec` with the profile's `CODEX_HOME` and the `-c` overrides `clauth start` already builds (`src/runtime.rs`). `AppConfig.profiles` stays claude-only; the MCP layer resolves a codex name through `CodexState` explicitly, so nothing else in the MCP surface changes.

**A result formatter is not the contract. The input and lifecycle contracts are.**
<!-- astra-objection-9: the earlier draft described only an output shim. Swapping the executable
     silently drops permission controls, the agent selection, resume-workspace resolution and the
     runtime teardown that the claude delegate carries today. -->

Every delegate option is **supported, translated, or refused by name** — never silently dropped. Flags below are from `codex exec --help` on codex 0.155, read on this host 2026-09-19.

| Option today | Codex arm | Note |
|---|---|---|
| `model` | translated → `-m/--model` | direct |
| `cwd` | translated → `-C/--cd` | direct |
| `env` | supported | layered the same way, then clauth's own keys win |
| `isolation` | supported | the same per-session home `clauth start --isolated` builds |
| `allowed_tools` | **refused** | codex has no per-tool allowlist. Dropping a permission control silently is the one failure this table exists to prevent |
| `permission_mode` | **refused pending a ruling** | codex's nearest are `-s/--sandbox {read-only,workspace-write,danger-full-access}` and `--approve-for-me`. The mapping is not one-to-one, so it needs an explicit ruling rather than a guess |
| `subagent_type` | **refused** | no codex equivalent |
| `resume` | **refused in this slice** | codex has `codex exec resume <id>`, but clauth's resume resolves the **workspace** from a claude transcript (`resolve_resume_workspace`, `src/mcp/mod.rs:3444-3452`). Two different session stores; a codex resume is its own PR |
| `args` (raw extra CLI arguments) | **translated with its own refusal list** | This row was missing from the first draft and it is the one that matters most: `args` is where a caller passes `--dangerously-skip-permissions` today, so it IS a permission control (`src/mcp/mod.rs:819-829`). The codex arm needs its own refusal list, because the claude one names claude flags (`--session-id`, `--resume`, `--fork-session`). At minimum it must refuse codex's sandbox-defeating flags — `--dangerously-bypass-approvals-and-sandbox` and `--dangerously-bypass-hook-trust` — unless the delegate contract is extended to carry that intent explicitly |
<!-- verify-P9b: the reviewer found `args` omitted from the table. Omitting it would have let a
     permission control through unexamined, which is precisely what this table exists to stop. -->

**Lifecycle, matched to the claude arm — but through the CODEX runtime type.** The claude delegate takes `ProfileRuntime::acquire` (`src/mcp/mod.rs:3464-3466`), whose acquisition builds Claude Code state. The codex arm must take **`CodexRuntime::acquire(name, isolation)`** instead — the type `clauth start` already uses for a codex session (`src/start.rs:750-753`, `src/runtime.rs:5477-5488`), with its own `Drop` teardown (`src/runtime.rs:5621`) — not a bare spawn and not the claude type.
<!-- verify-P9a: the first draft named `ProfileRuntime::acquire` for the codex arm. Wrong type;
     it would have built claude state for a codex run. --> Env composition goes through the **codex** engine, not `apply_delegate_env`, which pins `ClaudeEngine` and `CLAUDE_CONFIG_DIR` by construction (`src/mcp/mod.rs:3384-3403`): the codex twin scrubs `CODEX_MANAGED_ENV_KEYS` and pins `CODEX_HOME` (`src/harness.rs:149-186`). The `CLAUTH_MCP_DEPTH` recursion guard is set on the codex child too, so a nested `delegate` is refused there exactly as it is for claude. **Cancellation, a non-zero exit, and output produced before a failure each need a result shape written down in this slice, and this spec does not yet define them.** Naming them as "defined" without defining them is the gap the reviewer caught. The spec's position: the codex arm **inherits the claude arm's shapes verbatim** where they exist, and any case the claude arm does not already cover is an open item for the PR that implements G, listed in its description rather than discovered at review time.
<!-- verify-P9c: promising a shape is not specifying one. Inheriting explicitly, and naming the
     residue as open, is the honest version. -->

**Tests.** A shimmed `codex exec` produces the documented output; the formatter yields the same result keys as the claude formatter; a codex name that is quarantined is refused with the CLI's words; **each refused option is refused by name** with its reason, and none is silently dropped; the codex child carries `CODEX_HOME` and `CLAUTH_MCP_DEPTH` and none of `CODEX_MANAGED_ENV_KEYS` inherited; a cancelled run and a non-zero exit each return the defined shape; the runtime guard tears down on every exit path.

**Rulings wanted.** R10: which `codex exec` output contract to pin — `--json` (JSONL events, the closer twin of `--output-format stream-json`) versus `-o/--output-last-message <FILE>` (spec: `--json`). R10b: the `permission_mode` mapping onto `-s/--sandbox`, or keep refusing it. R11: whether `delegate` may run a codex profile that is not the active one (spec: yes, like `clauth start`).

## Delivery

Order A2, B, C, D, E, F, G. **A2 does not ship until R1-R3c are ruled on**, because it adds a program to the operator's `PATH` and that is not a decision to take on a fork's say-so. B carries the selection layer that C and D need, and its rename / disable invariants may split into their own PR (R4b). D owns the rollout-root enumeration that F reuses. E is a short slice once A2's key exists. F and G are independent of each other and of C-E.

Each PR: red tests first, the change, the wiki, the whole suite against a recorded baseline, and a cross-model review before it opens.

**What this spec has NOT done.** No slice is implemented. The withdrawn slice A had an implementation plan; it was never built, and the plan is marked withdrawn (`docs/superpowers/plans/2026-09-19-codex-follow-active.md`). The measured evidence here is **one rollout sample** on one Pro account and a read of `codex exec --help` on codex 0.155 — no poll fixture, and no runtime reproduction of any failure Astra described. Those are source-backed arguments, not runs.

## Astra review fold — an audit trail

Every item from the independent cross-model review (gpt-6-astra, high, read-only, 2026-09-19;
full text in `docs/superpowers/astra-spec-review-verdict.md`, verdict **DISAGREE**) and where it
landed. Each changed passage also carries an HTML comment naming its item, so the fold can be
audited in place.

| Item | Where it landed |
|---|---|
| Falsifying case + objection 1 (A's safety argument is false) | Slice A **withdrawn**. A2 replaces it, pinned decision 4 is rewritten, and the implementation plan is marked withdrawn |
| Objection 2 (operator slot vs managed session slot) | A2's in-session behavior, plus the `clauth_auth_store_owner` suffix-match weakness recorded as an open upstream question |
| Objection 3 (no state/link consistency or failure contract) | A2's "Failure and consistency contract" |
| Objection 4 (B mutates credentials with undefined invariants) | Slice B, rename and disable each specified; R4b offers to split them out |
| Objection 5 (selection identity across refreshes) | Architecture, the freeze-and-revalidate rules |
| Objection 6 (D and F scan the wrong roots; attribution undefined) | Slice D's root enumeration, deduplication and `unattributed` bucket; F inherits it |
| Objection 7 (D overgeneralizes one measurement) | "Measured tonight" rewritten to one sample; D labels windows by duration and distinguishes absent / un-run / failed |
| Objection 8 (C treats chain removal as deletion) | Slice C, marker preserved; R6b added |
| Objection 9 (G needs an input and lifecycle contract) | Slice G's option table and lifecycle paragraph |
| Contradictions (preserved rulings vs requested amendments) | The header paragraph, and the issue draft |
| Copy fix: threshold resets to 98, not clamped | Slice C tests |
| Copy fix: `c` already toggles cache counting on Tokens | Slice F, harness filter moved to `h` (R9b) |
| Copy fix: the adopt helper is not atomic on Windows | **Dissolved with slice A** — A2 calls no link helper |
| Copy fix: R3b's "new active" is misleading | **Dissolved with slice A** — that R3b is gone, and A2's R3b is a different question |
| Issue text: six corrections | Applied in `docs/codex-parity-issue-draft.md` |

### Second round: the repair-verification pass

A2 was new text no reviewer had seen, so it went back to the same reviewer for a **repair
verification** pass — one question per item, plus an adversarial pass on A2 alone. It returned
`REJECT`. Full text: `docs/superpowers/astra-fold-verification.md`. Every finding was confirmed
against source before it was acted on, and every repair is marked in place with a `verify-` comment.

| Finding | Confirmed how | Repair |
|---|---|---|
| **A2 forwarded `codex login` into the active profile's store** | `codex_spawn_command` appends arguments verbatim after pinning `CODEX_HOME` (`src/start.rs:559-573`) onto a home whose `auth.json` links to the profile store (`src/runtime.rs:5401-5405`) | The routed **allowlist**, with `login`/`logout` passed straight through (R3d) |
| "Either recursion guard alone is sufficient" was false | The baked path does not change where *clauth* looks; unix keeps the bare lookup (`src/runtime.rs:3560-3586`) | The env key is named load-bearing; the baked path is demoted to defense in depth |
| A symlink alias to the shim defeats a directory exclusion | Reasoning on the same lookup | Rejection by **canonical identity**, in generation and in the fallback |
| "on no PATH" too broad | `.../current@claude/bin` IS on a Claude Code session's PATH and NOT on a clean login shell's (both checked, 2026-09-19) | Stated precisely, with the design conclusion unchanged |
| Windows divergence described as automatic | `resolve_cli_command` scans matches in PATH order and prefers `.exe` (`src/runtime.rs:3575-3583`) | Described as configuration-dependent |
| "codex reads auth.json once" | codex reloads and checks the account matches | Reworded: the account is pinned, credentials can still refresh |
| Converge could recreate a shim the roster disowned | `CodexState::update` saves and releases (`src/codex_profiles.rs:191-199`) | Converge runs under the lock and re-reads the current key |
| Cursor identity only frozen at menu-open | `poll_codex_rows` replaces the vector regardless (`src/tui/app.rs:10135-10136`) | The cursor survives each replacement by name |
| Operator `archived_sessions` omitted | `CODEX_ROLLOUT_ROOTS = ["sessions", "archived_sessions"]` (`src/runtime.rs:5238-5247`) | Both operator roots listed; F inherits |
| Delegate named the wrong runtime type | `clauth start` uses `CodexRuntime::acquire` (`src/start.rs:750-753`) | Corrected to `CodexRuntime::acquire` |
| Delegate option table omitted raw `args` | `args` carries `--dangerously-skip-permissions` today (`src/mcp/mod.rs:819-829`) | Added, with its own refusal list |
| Result shapes promised but not defined | Read of the spec's own text | Says so, inherits the claude shapes, names the residue as open |
| Two lines still claimed "every ruling" is kept | Read of lines 3 and 15 | Both reworded |

**Not done, and worth saying:** these repairs have had **no third review**. Re-running a broad
discovery review after each repair round is the loop that exhausts a budget without improving the
artifact, so this spec stops here and says where it stopped. The A2 allowlist in particular is a
one-round-old design.

## Out of scope, kept from the parity map

Kick / auto-start (no codex endpoint), the spend ceiling and scoped weekly windows (no wire), per-session fallback and `--with-fallback` (codex binds `auth.json` at start), the macOS keychain mirror (forced file mode), `clauth proxy` (separate feature), settings sync and the sessions index (codex owns its own).
