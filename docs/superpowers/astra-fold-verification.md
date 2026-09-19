VERDICT: REJECT

Verified against `c05277151e710ee17aaf2ce6ce8f290c815274f9`. Read-only source analysis; no edits, tests, or `clauth` execution. The original live-adoption design is withdrawn correctly, but A2 introduces a credential-routing hazard and overstates its recursion protection.

## Item verification

- **Falsifying case — ANSWERED** (`docs/codex-parity-plan.md:94–114`; `docs/superpowers/plans/2026-09-19-codex-follow-active.md:3–11`).
- **Objection 1 — ANSWERED** (`docs/codex-parity-plan.md:47–49,210–214`; `src/actions.rs:1143–1156`).
- **Objection 2 — DISSOLVED.** The withdrawn operator-slot mutation has no referent in A2; the existing suffix-check weakness is explicitly retained as unresolved (`docs/codex-parity-plan.md:138–145`; `src/actions.rs:1573–1581`).
- **Objection 3 — PARTIAL.** The replacement consistency contract still permits a competing-writer failure: install commits `true`, uninstall commits `false` and removes the shim, then install’s delayed convergence recreates it (`docs/codex-parity-plan.md:198–205`; `src/codex_profiles.rs:191–199`). Specify serialization and a current-intent check covering convergence across CLI, TUI and doctor.
- **Objection 4 — ANSWERED** (`docs/codex-parity-plan.md:252–263`; `src/actions.rs:991–1015,1638–1650`).
- **Objection 5 — PARTIAL.** Freezing identity when a menu opens fixes subsequent reloads, but the text still does not preserve the selected profile across a reload **before** opening that menu, when the retained index can already name another account (`docs/codex-parity-plan.md:76–84`; `src/tui/app.rs:10135–10136`). Require identity preservation during roster replacement as well.
- **Objection 6 — PARTIAL.** Managed roots, attribution and isolated-session exclusions are addressed, but the operator root includes only `sessions`, omitting its `archived_sessions`; F inherits that omission (`docs/codex-parity-plan.md:297–304,324–326`; `src/runtime.rs:5238–5247`). Include operator archives or explicitly exclude them from coverage.
- **Objection 7 — ANSWERED** (`docs/codex-parity-plan.md:55,285,308`; `docs/codex-parity-issue-draft.md:36`).
- **Objection 8 — ANSWERED** (`docs/codex-parity-plan.md:271–276`; `src/tui/app.rs:6292–6303`; `src/fallback.rs:989–993`).
- **Objection 9 — PARTIAL.** The lifecycle paragraph incorrectly prescribes the same `ProfileRuntime` guard, whose acquisition builds Claude state; the Codex path uses `CodexRuntime::acquire` (`docs/codex-parity-plan.md:362`; `src/runtime.rs:2972–2976`; `src/start.rs:750–755`). The option table also omits raw `args`, including permission flags, and promises cancellation/error result shapes without actually defining or explicitly inheriting them (`docs/codex-parity-plan.md:351–364`; `src/mcp/mod.rs:819–829,3406–3413`).
- **Contradictions — PARTIAL.** The amendment paragraph and issue draft correctly distinguish requested changes, but the spec still says every slice keeps the recorded rulings and the chosen option keeps “every ruling” (`docs/codex-parity-plan.md:3,15`), contradicting its explicit disable/enable amendment at lines 5–10.
- **Copy fix: threshold resets to 98 — ANSWERED** (`docs/codex-parity-plan.md:276`; `src/codex_profiles.rs:132–135`; `src/profile.rs:1030`).
- **Copy fix: Tokens `c` collision — ANSWERED** (`docs/codex-parity-plan.md:330–336`; `src/tui/app.rs:3407–3410`).
- **Copy fix: Windows link replacement is not universally atomic — DISSOLVED.** A2 no longer uses the withdrawn link-replacement mechanism (`docs/codex-parity-plan.md:94–114`).
- **Copy fix: R3b’s “new active” — DISSOLVED.** That deletion proposal is withdrawn; the replacement R3b concerns command spelling (`docs/codex-parity-plan.md:240–244`).

## A2 adversarial pass

**1. Recursion: the ordinary path works only with the protections combined as specified; “either alone” is false.** Without the bypass key, the absolute clauth path still leads to `clauth → PATH codex → shim → absolute clauth`. Unix really does retain the bare lookup (`src/runtime.rs:3560–3586`); baking paths into the shim does not change clauth’s lookup.

With the bypass present and a genuine real-Codex target, re-entering the shim terminates correctly. However, excluding only the shim’s directory does not exclude a symlink in another PATH directory pointing back to the shim: baking that alias as `@REAL_CODEX@` makes the bypass branch repeatedly exec itself. This is a counterexample to both protections together unless generation **and stale-path fallback** reject executable aliases by resolved identity (`docs/codex-parity-plan.md:171–190`). The displayed shell also lacks the promised stale-path fallback.

**2. Shim directory: the plugin-tree correction is correct; the PATH claim needs qualification.** `expected_pointer()` names `<data_dir>/clauth/current@claude` as agentgear’s materialized plugin pointer (`src/plugin_host.rs:441–447`). Read-only inspection found `.claude-plugin/` and `hooks/`, with no `bin/`. However, this process’s PATH **does contain the nonexistent `current@claude/bin` entry**; “on no PATH” is too broad. Creating a sibling `<data_dir>/clauth/bin` remains consistent with the source.

**3. Windows: the resolver preference is confirmed, but the shell comparison is conditional.** The resolver scans all matches and chooses an `.exe` before falling back to the first match (`src/runtime.rs:3575–3583`). Divergence is possible with a `.cmd` shim in an earlier PATH directory and an `.exe` later; it is not inevitable merely because both exist “beside” each other. Windows refusal is a defensible scope choice, but its explanation should describe that configuration; I did not execute Windows resolution.

**4. Argument forwarding and profile home: confirmed for the existing named-start path.** The parser preserves the trailing argument vector, `cmd_start` passes it to `run_codex`, and the command builder appends it unchanged after clauth’s two configuration overrides (`src/cli.rs:465–471,511–517`; `src/main.rs:468`; `src/start.rs:559–572`). `CODEX_HOME` is pinned to the acquired Codex runtime home (`src/start.rs:750–755`), whose auth link targets that profile’s store (`src/runtime.rs:5383,5401–5405`). The proposed `--codex-active` parser path is not implemented or verified.

**5. Next-launch boundary: correct for account selection, with misleading supporting wording.** The `--with-fallback` refusal really states that boundary (`src/main.rs:453–458`), and switching only changes the marker (`src/actions.rs:1143–1156`). But “reads auth.json once” is not literally correct: Codex reloads credentials and checks that the account identity matches. Say that **account selection remains fixed while same-account credentials can refresh**. Nested launches taking A2’s bypass intentionally retain their inherited home rather than consume the new marker. See [Codex 0.155 `manager.rs`, `reload_if_account_id_matches` and `refresh_token`](https://raw.githubusercontent.com/openai/codex/rust-v0.155.0/codex-rs/login/src/auth/manager.rs).

**6. New credential hazard: ordinary login commands now target a managed profile implicitly.** Consider an active clauth profile A and an operator login that was **never adopted**. Slice A left that operator login alone; A2 forwards every `codex <args>` through A (`docs/codex-parity-plan.md:126–130`). Consequently, `codex login --with-api-key` receives A’s runtime home and forced file backend, then truncates and replaces A’s linked credential store. The call chain is confirmed by `src/start.rs:567–572`, `src/runtime.rs:5401–5405`, [Codex `cli/src/login.rs:189–209`](https://raw.githubusercontent.com/openai/codex/rust-v0.155.0/codex-rs/cli/src/login.rs), and [Codex file storage’s `save` implementation](https://raw.githubusercontent.com/openai/codex/rust-v0.155.0/codex-rs/login/src/auth/storage.rs).

That defeats A2’s blanket credential-safety claim even though the shim itself performs no credential write. Specify which subcommands select a managed account and an explicit policy for credential-mutating commands before accepting A2.

These are source-backed counterexamples, not runtime reproductions. I did not independently remeasure the reported rollout sample.
