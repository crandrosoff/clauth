VERDICT: DISAGREE

## Slice A: the falsifying case

**The ordinary cross-account switch already breaks the promised behavior, without a race.** Start plain Codex with the adopted link pointing to account A. Switch clauth to B. When Codex next refreshes, `reload_if_account_id_matches` compares the newly loaded account with cached A. B fails that check; `refresh_token` returns a permanent account-mismatch error instead of adopting B. This is explicit in [Codex 0.155’s auth manager](https://raw.githubusercontent.com/openai/codex/rust-v0.155.0/codex-rs/login/src/auth/manager.rs), in `reload_if_account_id_matches` and `refresh_token`.

The spec’s “uses what it finds” at `docs/codex-parity-plan.md:74` mistakes **refreshing credentials for the same account** for **changing accounts**. The older plan’s reload summary at `docs/codex-plan.md:43` omits that crucial qualification.

**There is also a more damaging in-flight-refresh scenario:**

1. The operator link points to A. Codex completes its guarded reload and submits A’s refresh token.
2. Before the response arrives, `follow_active` repoints the link to B.
3. Codex receives A’s replacement tokens. `persist_tokens` reloads storage through the now-repointed path, merges those tokens into that document, and saves it.
4. B’s store receives A’s tokens; A’s store retains the spent token. A subsequent clauth refresh of A can replay it.

That interleaving follows [Codex 0.145’s `persist_tokens` and refresh implementation](https://raw.githubusercontent.com/openai/codex/rust-v0.145.0/codex-rs/login/src/auth/manager.rs) and its [path-based, truncate-and-write file storage](https://raw.githubusercontent.com/openai/codex/rust-v0.145.0/codex-rs/login/src/auth/storage.rs). Locally, `docs/codex-plan.md:39–43` documents both in-place writes and the remaining TOCTOU window; `src/codex_auth.rs:918–964` shows clauth subsequently reading and refreshing A’s unchanged store.

Atomic replacement of the **symlink** does not make the **read → network refresh → persistence** transaction atomic. The spec’s open-file explanation does not protect a later reopen for persistence.

These are source-backed failure scenarios, not runtime reproductions. I did not run `clauth`.

## Material objections (ranked, most severe first)

1. **A’s central safety argument is false.**  
   **Where:** Slice A, especially lines 74–85.  
   **Why:** Both failures above defeat the feature’s purpose. “One physical file with one refresher” also misstates the architecture: Codex and clauth can both refresh. Cl auth’s stand-down depends on `has_live_session` (`src/codex_auth.rs:894`), which checks clauth session markers (`src/runtime.rs:635`); a standalone operator Codex supplies no such marker. That limitation predates A, but A cannot use the guard as proof that its new switching behavior is safe.  
   **Change:** Withdraw live adoption as a supported behavior. Prefer selecting the account at launch with a stable per-session home. Retaining mutable operator links requires an actual reader/writer coordination design; a clauth-only rotation lock or “next launch only” wording does not prevent an already-running bare Codex from writing through the link. Require account-mismatch and delayed-refresh-response cases before approving A.

2. **“Operator slot” is not sufficiently distinguished from a managed session slot.**  
   **Where:** Slice A steps 1–2.  
   **Why:** Inside `clauth start A`, inherited `CODEX_HOME/auth.json` is itself an adopted-looking link to A’s store. Following the specified algorithm when switching to B repoints that running session’s private link while its registry still identifies A. Capture explicitly rejects this environment in [src/actions.rs:1551](/home/rocky00717/rawgentic/projects/clauth/.worktrees/parity-plan/src/actions.rs:1551). Also, manual CLI and daemon processes can have different `CODEX_HOME` values, so the same setting can control different files.  
   **Change:** Define a stable operator-home policy shared by CLI and scheduler, explicitly excluding managed session homes. Specify canonical ownership checks. The existing `clauth_auth_store_owner` at line 1573 checks only the suffix `profiles/<name>/auth.json`, not membership under the actual clauth root; reusing it would violate “link elsewhere → untouched.”

3. **A lacks a state/link consistency and failure contract.**  
   **Where:** Slice A mechanism, tests, and “one function” claim.  
   **Why:** `set_active` only mutates memory; `CodexState::update` saves afterward (`src/codex_profiles.rs:191–199`). Repointing after the update returns permits competing switches to leave the marker and link naming different profiles. Repointing inside the closure leaves a changed link if state persistence fails. The adoption helper returns `false` on failure, while the proposed log unconditionally announces success. An already-active switch currently returns early (`src/actions.rs:1151`), defeating repair after partial failure. Finally, `Off` bypasses `switch_codex_profile` entirely (`src/usage/scheduler.rs:3637`).  
   **Change:** Specify serialization, partial-failure reporting, retry/reconciliation behavior, and already-active reconciliation. Cover `Off` explicitly and derive its retained-account message from the actual link, not the previous marker. Add failure and competing-switch tests; filesystem happy paths are insufficient.

4. **B introduces credential mutations without defining their invariants.**  
   **Where:** Slice B mechanism and R5.  
   **Why:** There is no existing codex rename writer to route to: `rename_profile` takes `AppConfig` and operates on Claude state (`src/actions.rs:991`). Renaming an adopted codex profile also needs to preserve or explicitly detach the operator link; moving the directory alone strands it. For disable, skipping usage and chain selection is only part of Claude semantics: active/live profiles are refused (`src/actions.rs:1638–1650`), and operational entry points reject disabled targets.  
   **Change:** Specify codex rename’s roster, directory, link, rotation-guard and recovery behavior. Define disable across manual switch, start, refresh and later delegate, including active/live refusal. Otherwise narrow B’s initial action set.

5. **The new selection layer needs identity preservation across refreshes.**  
   **Where:** `RowSel::Codex(usize)` and the hot-reload paragraph.  
   **Why:** `poll_codex_rows` replaces the entire vector every second, even with a modal open (`src/tui/app.rs:10127–10136`). A CLI deletion or reorder can make index 1 identify a different account between selection and action. Adding the harness tag prevents cross-harness mistakes, but does not prevent wrong-account mutations.  
   **Change:** Preserve selection by profile identity across reloads, represent an empty selection, and freeze the target’s harness/name when opening an action or confirmation. Revalidate that target before mutation. Test reorder/delete while the menu is open.

6. **D and F scan the wrong primary store and leave account attribution undefined.**  
   **Where:** Slice D’s supplement and Slice F’s feeder.  
   **Why:** `clauth start` rollouts live under `~/.clauth/profiles/<name>/codex-home/sessions`; isolated rollouts live in temporary session homes and are discarded (`wiki/Codex.md:59–69`). Scanning only `~/.codex/sessions` misses the sessions this integration launches. Conversely, an operator directory can contain multiple accounts’ history; a recent event plus a live profile does not establish ownership.  
   **Change:** Enumerate supported roots, including custom operator homes, and define deduplication, archive handling and isolated-session coverage. Require defensible account attribution before a rollout supplements a profile’s usage. Do not infer historical ownership from today’s active marker or auth link.

7. **D overgeneralizes the measurement and conflates two data sources.**  
   **Where:** “Measured tonight,” Slice D behavior/docs/tests.  
   **Why:** One observed rollout’s `primary.window_minutes = 10080` does not establish that every Pro/Business account—or its `wham/usage` response—has only a weekly window. “One window” does not inherently mean “weekly.” The existing contract maps by duration, not position or plan name (`docs/codex-plan.md:44,205,212`).  
   **Change:** Label windows from their actual duration. Distinguish absent data from a successfully observed one-window response. Keep the plan claim scoped to the measured sample and obtain a corresponding poll fixture before declaring the existing 5h dash proven correct.

8. **C treats removing chain membership as deleting a profile.**  
   **Where:** Slice C tests, line 111.  
   **Why:** `CodexState::remove_profile` clears the active marker when deleting the roster entry (`src/codex_profiles.rs:168–175`). That is not a chain-editing precedent. The existing Claude chain editor removes membership without clearing the marker (`src/tui/app.rs:6292–6303`); the codex walker simply stops when the active profile is outside the chain (`src/fallback.rs:989–993`).  
   **Change:** Preserve the marker on chain removal, or request an explicit new ruling. Do not describe clearing it as existing CLI semantics.

9. **G needs an input and lifecycle contract, not only an output formatter.**  
   **Where:** Slice G mechanism and R10.  
   **Why:** The current delegate handles Claude-specific permissions, allowed tools, agents, resume, environment composition and runtime teardown (`src/mcp/mod.rs:3377–3403,3441–3466,3523–3545`). Swapping the executable and matching result keys does not preserve those semantics. Silently dropping permission controls would be especially material.  
   **Change:** Enumerate supported, translated and refused delegate options. Require the codex runtime acquisition/spawn/teardown path and specify cancellation, partial-output and error behavior. Add option-refusal and lifecycle tests alongside the output shim.

## Contradictions with codex-plan.md rulings

- **Switch semantics are being amended.** Original decision 4: “A codex switch writes only `codex-profiles.toml`” (`docs/codex-plan.md:21`). Shipped deviation: “the slot IS the switch and it lands at the next codex session” (line 228). New Slice A: “also repoints the operator’s own login file” and “A running codex adopts the new chain” (line 74). Default-off preserves default behavior; it does not make this an unchanged ruling. Request explicit approval for the new semantics, after correcting the safety premise.

- **Disable/enable deliberately reverse a settled CLI restriction.** Settled question 8 says those verbs “refuse a codex name as a codex profile” (`docs/codex-plan.md:217`). Slice B “adds them to `CodexState` first … and to the CLI verbs” (line 99). R4 acknowledges the proposed extension, but the introductory claim that nothing reverses a #69 decision remains inaccurate.

- **F and delegate G are not contradictions.** The original explicitly calls the codex cost lens “its own feature” (line 218) and `delegate` “its own PR after this series” (line 188). Likewise, adding a Usage view need not reverse the named-window model if it continues using duration-mapped cache fields.

## Issue text

It accurately summarizes much of the proposed scope, but I would not send it unchanged.

- Line 11 repeats A’s false live-adoption promise and understates it as a small atomic-link change. Lead with the unresolved account-binding and refresh-transaction problem.
- “Each keeping #69’s rulings” should distinguish preserved decisions from requested amendments.
- “Rulings … before B onward” is inconsistent with R1–R3 governing A itself. Obtain those before A. Include the omitted R3b and R5.
- “I have this one built on a branch” is not substantiated by this reviewed checkout: its diff from `37b4ca08` contains documentation only. Supply the implementation branch/commit and accurately bounded validation evidence.
- “Link below” has no link.
- The Pro measurement and rollout-feed claims need the qualifications above.

## Copy fixes

- Slice C: invalid file thresholds reset to **98**, rather than clamp to 50–100 (`src/codex_profiles.rs:127–135`).
- Slice F: `c` already toggles cache counting on Tokens (`src/tui/app.rs:3403–3409`); choose or explicitly reassign a key.
- Slice A: the helper is not universally atomic; Windows uses remove-then-rename (`src/actions.rs:1599–1603`).
- R3b’s “new active” is misleading: deleting the active codex profile clears the marker; it does not select a replacement.
