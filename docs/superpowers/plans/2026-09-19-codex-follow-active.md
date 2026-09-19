# `follow_active` (codex parity slice A) Implementation Plan

> **WITHDRAWN 2026-09-19 (decisions D11, D12).** Do not implement this plan. Astra
> (gpt-6-astra, high, read-only) falsified the design with codex 0.155's own source:
> `reload_if_account_id_matches` makes a running codex REFUSE a reloaded `auth.json`
> that names a different account, and codex persists refreshed tokens by PATH, so
> repointing the link during an in-flight refresh can write account A's new tokens
> into account B's store. Verdict: `docs/superpowers/astra-spec-review-verdict.md`.
> Slice A is replaced by **A2, launch-time account selection**, in
> `docs/codex-parity-plan.md`. This file stands only as the record of the withdrawn
> design.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** With `follow_active = true` in `codex-profiles.toml`, every codex switch (manual `clauth <name>` or the chain's auto-switch) also repoints the operator's own `~/.codex/auth.json` link onto the new active profile's store, so a plain `codex` starts on the chain's pick.

**Architecture:** One new boolean on `CodexState`; one new private function in `src/actions.rs` that inspects the operator slot and repoints it through the existing tmp-sibling-then-rename helper; one call site in `switch_codex_profile`, which both switch paths already use. The switch's return value gains the path it repointed so each caller can print or log its own line.

**Tech Stack:** Rust 2024, serde + toml, the crate's own `HomeSandbox` inline tests (`cargo test --bin clauth -- <filter>`).

**Spec:** `docs/codex-parity-plan.md`, section "A. `follow_active`".

## Global Constraints

- Default is `false`; an absent key reads as `false` and is never invented into the file on save (`skip_serializing_if`), matching `weekly_switch_threshold`'s handling.
- Never touch an operator slot that is not a symlink into `~/.clauth/profiles/<any>/auth.json`. A regular file, an absent file, or a link elsewhere is left exactly as it is.
- Never fail a switch because the follow could not happen: the marker move is the switch; the follow is best-effort and reports what it did.
- No new copy of any chain is ever created (codex-plan.md decision 8).
- Wiki copy uses the CLI's own message style: `clauth: <path> now follows '<name>'`.
- Whole suite baseline on this host: `3902 passed; 2 failed; 1 ignored` (the 2 are umask 0002 artifacts; they pass under `umask 022`).

---

### Task 1: the `follow_active` key on `CodexState`

**Files:**
- Modify: `src/codex_profiles.rs:62-80` (the struct), plus a getter beside `active_profile()` (~line 111)
- Test: `tests/inline/codex_profiles.rs` (append)

**Interfaces:**
- Produces: `CodexState::follow_active(&self) -> bool`

- [ ] **Step 1: Write the failing tests**

```rust
/// `follow_active` reads `true` only when the file says so; an absent key is
/// `false`, and a save never invents the key into a file that lacked it.
#[test]
fn follow_active_defaults_off_and_round_trips() {
    let _home = crate::testutil::HomeSandbox::new();
    let dir = crate::profile::clauth_dir().expect("clauth dir");
    crate::profile::mkdir_700(&dir).expect("mkdir");
    std::fs::write(dir.join("codex-profiles.toml"), "profiles = [\"cx\"]\n").expect("write");
    let state = crate::codex_profiles::CodexState::load().expect("load");
    assert!(!state.follow_active(), "absent key reads false");

    // A save of an untouched file must not add the key.
    crate::codex_profiles::CodexState::update(|st| { st.set_active(Some("cx")); Ok(()) }).expect("update");
    let body = std::fs::read_to_string(dir.join("codex-profiles.toml")).expect("read");
    assert!(!body.contains("follow_active"), "save invented the key: {body}");

    std::fs::write(dir.join("codex-profiles.toml"), "profiles = [\"cx\"]\nfollow_active = true\n").expect("write");
    let state = crate::codex_profiles::CodexState::load().expect("load");
    assert!(state.follow_active(), "the key reads true");
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test --bin clauth -- follow_active_defaults_off_and_round_trips`
Expected: FAIL to compile — `no method named follow_active`.

- [ ] **Step 3: Add the field and the getter**

In the struct (after `weekly_switch_threshold`):

```rust
    /// Slice A of the parity series: when true, a codex switch also repoints
    /// the operator's own `auth.json` link onto the new active profile's
    /// store, so a plain `codex` starts on the chain's pick. Off by default;
    /// an absent key is never written back (same rule as the weekly line).
    #[serde(default, skip_serializing_if = "std::ops::Not::not")]
    follow_active: bool,
```

Beside `active_profile()`:

```rust
    /// Whether a switch repoints the operator's codex login link (slice A).
    pub(crate) fn follow_active(&self) -> bool {
        self.follow_active
    }
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `cargo test --bin clauth -- follow_active_defaults_off_and_round_trips`
Expected: `1 passed`.

- [ ] **Step 5: Commit**

```bash
git add src/codex_profiles.rs tests/inline/codex_profiles.rs
git commit -m "feat(codex): the follow_active key on the codex roster, off by default"
```

### Task 2: `follow_operator_slot` — inspect and repoint the operator's link

**Files:**
- Modify: `src/actions.rs` — new function near `adopt_operator_auth_slot` (~line 1588); hoist nothing else
- Test: `tests/inline/actions.rs` (append, after the codex CRUD tests ~line 1412)

**Interfaces:**
- Consumes: `adopt_operator_auth_slot(auth_path: &Path, store: &Path) -> bool` (exists, private), `profile_dir(&ProfileName) -> Result<PathBuf>` (exists), `crate::runtime::is_codex_home_path(&Path) -> bool` (exists), `default_codex_operator_home() -> Result<PathBuf>` (exists).
- Produces:

```rust
/// What the follow did to the operator's codex login slot.
#[derive(Debug, PartialEq, Eq)]
pub(crate) enum Followed {
    /// The slot was a link into the profile store and now points at `name`'s chain.
    Repointed(std::path::PathBuf),
    /// The slot already pointed at `name`'s chain; nothing was touched.
    Already,
    /// The slot is not a link into the profile store (a real file, absent,
    /// or a link elsewhere), or lives inside a clauth session home; untouched.
    NotAdopted,
}

pub(crate) fn follow_operator_slot(name: &str) -> Followed
```

- [ ] **Step 1: Write the failing tests**

```rust
// ── follow_active: the operator's codex link follows a switch ──────────────

/// Lays `<home>/.clauth/profiles/<name>/auth.json` and returns its path.
fn seed_codex_store(name: &str) -> std::path::PathBuf {
    let dir = crate::profile::profile_dir(&crate::profile::ProfileName::from(name)).expect("profile dir");
    crate::profile::mkdir_700(&dir).expect("mkdir");
    let store = dir.join("auth.json");
    std::fs::write(&store, "{}").expect("store");
    store
}

/// The operator slot under the sandbox home, `~/.codex/auth.json`.
fn operator_slot() -> std::path::PathBuf {
    let home = crate::profile::home_dir().expect("home");
    let dir = home.join(".codex");
    std::fs::create_dir_all(&dir).expect("codex home");
    dir.join("auth.json")
}

#[cfg(unix)]
#[test]
fn follow_repoints_an_adopted_link_onto_the_new_store() {
    let _home = HomeSandbox::new();
    let cx1 = seed_codex_store("cx1");
    let cx2 = seed_codex_store("cx2");
    let slot = operator_slot();
    std::os::unix::fs::symlink(&cx1, &slot).expect("adopted link");

    assert_eq!(follow_operator_slot("cx2"), Followed::Repointed(slot.clone()));
    assert_eq!(std::fs::read_link(&slot).expect("link"), cx2, "the link now names cx2's chain");

    assert_eq!(follow_operator_slot("cx2"), Followed::Already, "a second follow is a no-op");
    assert_eq!(std::fs::read_link(&slot).expect("link"), cx2);
}

#[cfg(unix)]
#[test]
fn follow_leaves_an_unadopted_slot_alone() {
    let _home = HomeSandbox::new();
    seed_codex_store("cx2");
    let slot = operator_slot();

    // Absent.
    assert_eq!(follow_operator_slot("cx2"), Followed::NotAdopted);
    assert!(!slot.exists(), "nothing was created");

    // A real file: the operator's own login, never hijacked.
    std::fs::write(&slot, "{\"mine\":true}").expect("own login");
    assert_eq!(follow_operator_slot("cx2"), Followed::NotAdopted);
    assert_eq!(std::fs::read_to_string(&slot).expect("read"), "{\"mine\":true}");

    // A link somewhere else entirely.
    std::fs::remove_file(&slot).expect("rm");
    let elsewhere = crate::profile::home_dir().expect("home").join("elsewhere.json");
    std::fs::write(&elsewhere, "{}").expect("elsewhere");
    std::os::unix::fs::symlink(&elsewhere, &slot).expect("foreign link");
    assert_eq!(follow_operator_slot("cx2"), Followed::NotAdopted);
    assert_eq!(std::fs::read_link(&slot).expect("link"), elsewhere);
}

/// `CODEX_HOME` set to a clauth session home means the caller runs inside a
/// `clauth start` session; the operator's slot is not reachable there, and the
/// follow must not fail the switch — it reports NotAdopted.
#[cfg(unix)]
#[test]
fn follow_inside_a_session_home_is_a_no_op() {
    let _home = HomeSandbox::new();
    seed_codex_store("cx2");
    let dir = crate::profile::profile_dir(&crate::profile::ProfileName::from("cx1")).expect("dir");
    let session_home = dir.join("codex-home-abc");
    std::fs::create_dir_all(&session_home).expect("session home");
    let _env = crate::testutil::EnvVarGuard::set("CODEX_HOME", &session_home);
    assert_eq!(follow_operator_slot("cx2"), Followed::NotAdopted);
}
```

If `crate::testutil::EnvVarGuard` does not exist, look for the env guard the codex login tests use (`grep -n "CODEX_HOME" tests/inline/codex_login.rs`) and use that helper; do not write a new one.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cargo test --bin clauth -- follow_`
Expected: FAIL to compile — `cannot find function follow_operator_slot`, `cannot find type Followed`.

- [ ] **Step 3: Implement the function**

Place it directly above `adopt_operator_auth_slot` in `src/actions.rs`:

```rust
/// What the follow did to the operator's codex login slot.
#[derive(Debug, PartialEq, Eq)]
pub(crate) enum Followed {
    Repointed(std::path::PathBuf),
    Already,
    NotAdopted,
}

/// Slice A of the parity series: after a codex switch, make the operator's
/// own `auth.json` — `$CODEX_HOME/auth.json`, default `~/.codex/auth.json` —
/// follow the new active profile, so a plain `codex` starts on the chain's
/// pick. Only a slot that is ALREADY a link into the profile store moves:
/// a real file is the operator's own login and is never hijacked, an absent
/// file is never invented, and a link elsewhere is somebody else's business.
/// A `CODEX_HOME` inside a clauth session home means the caller runs inside
/// `clauth start`, where the operator slot is not reachable: no-op, never an
/// error, because the marker move is the switch and this is the courtesy
/// after it. Repointing goes through the same tmp-sibling-then-rename the
/// capture uses, so a codex reading the old chain keeps its open file until
/// its next reload-before-refresh, which is the documented boundary.
pub(crate) fn follow_operator_slot(name: &str) -> Followed {
    let home = match std::env::var_os("CODEX_HOME").filter(|d| !d.is_empty()) {
        Some(dir) => {
            let dir = std::path::PathBuf::from(dir);
            if crate::runtime::is_codex_home_path(&dir) {
                return Followed::NotAdopted;
            }
            dir
        }
        None => match default_codex_operator_home() {
            Ok(dir) => dir,
            Err(_) => return Followed::NotAdopted,
        },
    };
    let slot = home.join("auth.json");
    let Ok(target) = std::fs::read_link(&slot) else {
        return Followed::NotAdopted;
    };
    let Ok(profiles_root) = crate::profile::clauth_dir().map(|d| d.join("profiles")) else {
        return Followed::NotAdopted;
    };
    // A link into the store looks like <root>/<some-name>/auth.json exactly.
    let in_store = target.file_name().is_some_and(|f| f == "auth.json")
        && target.parent().and_then(|p| p.parent()) == Some(profiles_root.as_path());
    if !in_store {
        return Followed::NotAdopted;
    }
    let Ok(store) = profile_dir(&ProfileName::from(name)).map(|d| d.join("auth.json")) else {
        return Followed::NotAdopted;
    };
    if target == store {
        return Followed::Already;
    }
    if adopt_operator_auth_slot(&slot, &store) {
        Followed::Repointed(slot)
    } else {
        Followed::NotAdopted
    }
}
```

`default_codex_operator_home` and `profile_dir` already exist in this file; if `profile_dir` is imported under another name, use that name.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cargo test --bin clauth -- follow_`
Expected: `3 passed`.

- [ ] **Step 5: Commit**

```bash
git add src/actions.rs tests/inline/actions.rs
git commit -m "feat(codex): follow_operator_slot repoints an adopted login link onto a profile's store"
```

### Task 3: wire the follow into `switch_codex_profile`, both callers report it

**Files:**
- Modify: `src/actions.rs:1143-1157` (`switch_codex_profile`)
- Modify: `src/main.rs:~1686-1690` (`cmd_switch`'s codex arm)
- Modify: `src/usage/scheduler.rs:~3629-3636` (the `SwitchAction::To` arm)
- Test: `tests/inline/actions.rs` (append)

**Interfaces:**
- Produces: `switch_codex_profile(name: &str) -> Result<Option<std::path::PathBuf>>` — `Some(path)` when the operator slot was repointed, else `None`. Existing callers that `.expect("switch")` keep compiling.

- [ ] **Step 1: Write the failing tests**

```rust
#[cfg(unix)]
#[test]
fn switch_codex_follows_the_operator_link_only_when_the_key_is_on() {
    let _home = HomeSandbox::new();
    let cx1 = seed_codex_store("cx1");
    let cx2 = seed_codex_store("cx2");
    let slot = operator_slot();
    std::os::unix::fs::symlink(&cx1, &slot).expect("adopted link");

    // Key off: the marker moves, the link does not.
    write_codex_state("active_profile = \"cx1\"\nprofiles = [\"cx1\", \"cx2\"]\n");
    assert_eq!(switch_codex_profile("cx2").expect("switch"), None);
    assert_eq!(std::fs::read_link(&slot).expect("link"), cx1, "key off leaves the link");

    // Key on: both move.
    write_codex_state("active_profile = \"cx1\"\nprofiles = [\"cx1\", \"cx2\"]\nfollow_active = true\n");
    assert_eq!(switch_codex_profile("cx2").expect("switch"), Some(slot.clone()));
    assert_eq!(std::fs::read_link(&slot).expect("link"), cx2, "key on repoints the link");

    // Already active: a no-op switch touches nothing and reports nothing.
    assert_eq!(switch_codex_profile("cx2").expect("no-op"), None);
    assert_eq!(std::fs::read_link(&slot).expect("link"), cx2);
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test --bin clauth -- switch_codex_follows_the_operator_link_only_when_the_key_is_on`
Expected: FAIL — the return type is `()`, so `assert_eq!(…, None)` does not compile.

- [ ] **Step 3: Change `switch_codex_profile`**

```rust
/// Move the codex active marker to `name`. Returns the operator slot that
/// now follows it when `follow_active` is on and the slot was an adopted
/// link (slice A), else `None`. The marker is the switch; the follow is the
/// courtesy after it and never fails the call.
pub(crate) fn switch_codex_profile(name: &str) -> Result<Option<std::path::PathBuf>> {
    let (changed, follow) = crate::codex_profiles::CodexState::update(|state| {
        if !state.holds(name) {
            bail!("codex profile '{name}' not found");
        }
        crate::codex_auth::refuse_if_quarantined(name)?;
        if state.active_profile().map(ProfileName::as_str) == Some(name) {
            return Ok((false, false));
        }
        state.set_active(Some(name));
        Ok((true, state.follow_active()))
    })?;
    if !(changed && follow) {
        return Ok(None);
    }
    Ok(match follow_operator_slot(name) {
        Followed::Repointed(path) => Some(path),
        Followed::Already | Followed::NotAdopted => None,
    })
}
```

Keep whatever the existing body does between `refuse_if_quarantined` and `set_active` (lines 1149-1150 hold a comment or a check today — read them and keep them).

- [ ] **Step 4: Report it in both callers**

`src/main.rs`, the codex arm of `cmd_switch`:

```rust
        if let Some(canonical) = codex_profiles::CodexState::load()?.canonical_name(name) {
            let followed = actions::switch_codex_profile(&canonical)?;
            outln!("clauth: switched codex to '{canonical}'");
            if let Some(path) = followed {
                outln!("clauth: {} now follows '{canonical}'", path.display());
            }
            return Ok(());
        }
```

`src/usage/scheduler.rs`, the `SwitchAction::To` arm:

```rust
        crate::fallback::SwitchAction::To(target) => match crate::actions::switch_codex_profile(target.as_str()) {
            Err(e) => logline!("clauth: codex auto-switch to '{target}' failed: {e:#}"),
            Ok(Some(path)) => logline!(
                "clauth: codex auto-switched to '{target}' — {} follows it, live at the next codex start or reload",
                path.display()
            ),
            Ok(None) => logline!(
                "clauth: codex auto-switched to '{target}' — live at the next codex session"
            ),
        },
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `cargo test --bin clauth -- codex`
Expected: every codex test passes, including the five existing `switch_codex_profile` tests (their `.expect("switch")` still compiles against `Result<Option<PathBuf>>`).

- [ ] **Step 6: Commit**

```bash
git add src/actions.rs src/main.rs src/usage/scheduler.rs tests/inline/actions.rs
git commit -m "feat(codex): a codex switch follows the operator link when follow_active is on"
```

### Task 4: the auto-switch path, and `Off` leaves the link

**Files:**
- Test: `tests/inline/scheduler.rs` (append after `apply_codex_switch_moves_the_on_disk_marker`, ~line 10948)

**Interfaces:**
- Consumes: `seed_codex_walk`, `third_party_state`, `codex_active`, `CODEX_SPENT`, `CODEX_IDLE`, `REFRESH_INTERVAL_MS`, `super::apply_codex_switch` — all exist in that test file.

- [ ] **Step 1: Write the failing test**

```rust
/// Slice A through the chain: with `follow_active = true` the auto-switch
/// repoints the operator's adopted link onto the member it walked to, and a
/// wrap-off that clears the slot leaves the link where it is — a cleared
/// marker must not strand a working login.
#[cfg(unix)]
#[test]
fn apply_codex_switch_follows_the_operator_link_and_off_leaves_it() {
    let _home = crate::testutil::HomeSandbox::new();
    let state = third_party_state(crate::providers::fetch_third_party_usage);
    let home = crate::profile::home_dir().expect("home");
    let profiles = crate::profile::clauth_dir().expect("clauth").join("profiles");
    for n in ["cx1", "cx2"] {
        std::fs::create_dir_all(profiles.join(n)).expect("dir");
        std::fs::write(profiles.join(n).join("auth.json"), "{}").expect("store");
    }
    std::fs::create_dir_all(home.join(".codex")).expect("codex home");
    let slot = home.join(".codex/auth.json");
    std::os::unix::fs::symlink(profiles.join("cx1/auth.json"), &slot).expect("adopted");

    let codex = seed_codex_walk(
        &state,
        "active_profile = \"cx1\"\nprofiles = [\"cx1\", \"cx2\"]\nfallback_chain = [\"cx1\", \"cx2\"]\nfollow_active = true\n",
        &[("cx1", CODEX_SPENT), ("cx2", CODEX_IDLE)],
    );
    super::apply_codex_switch(&state, &codex, REFRESH_INTERVAL_MS);
    assert_eq!(codex_active().as_deref(), Some("cx2"));
    assert_eq!(std::fs::read_link(&slot).expect("link"), profiles.join("cx2/auth.json"));

    let codex = seed_codex_walk(
        &state,
        "active_profile = \"cx2\"\nprofiles = [\"cx1\", \"cx2\"]\nfallback_chain = [\"cx1\", \"cx2\"]\nwrap_off = true\nfollow_active = true\n",
        &[("cx1", CODEX_SPENT), ("cx2", CODEX_SPENT)],
    );
    super::apply_codex_switch(&state, &codex, REFRESH_INTERVAL_MS);
    assert_eq!(codex_active(), None, "every member spent: switched off");
    assert_eq!(std::fs::read_link(&slot).expect("link"), profiles.join("cx2/auth.json"), "off leaves the link");
}
```

If `seed_codex_walk` writes the store files itself, drop the loop that writes them; read its body first (`grep -n "fn seed_codex_walk" tests/inline/scheduler.rs`).

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test --bin clauth -- apply_codex_switch_follows_the_operator_link_and_off_leaves_it`
Expected: the first half passes only if Task 3 landed; run it before Task 3 to see it FAIL on the `read_link` assertion (the link still names cx1), then again after.

- [ ] **Step 3: Run it to verify it passes**

Run: `cargo test --bin clauth -- apply_codex_switch`
Expected: `3 passed`.

- [ ] **Step 4: Commit**

```bash
git add tests/inline/scheduler.rs
git commit -m "test(codex): the auto-switch follows the operator link, wrap-off leaves it"
```

### Task 5: wiki copy

**Files:**
- Modify: `wiki/Codex.md` — the "Switch" section (~line 84) and the "Auto-switch" section (~line 107)
- Modify: `wiki/Configuration.md` — the `codex-profiles.toml` table (~line 172-178)

- [ ] **Step 1: Switch section** — after "A codex switch moves the active marker in `codex-profiles.toml` and nothing else", add:

> Set `follow_active = true` in that file and a switch also repoints your own `~/.codex/auth.json` (or `$CODEX_HOME/auth.json`) onto the new profile's store, so a plain `codex` starts on the account you switched to; the CLI says `clauth: /home/you/.codex/auth.json now follows 'work'`. Only a slot that is already a link into `~/.clauth/profiles/` moves: your own login file, an unadopted copy, or a link elsewhere is left alone. A codex already running keeps the account it started on until its next reload-before-refresh, when it picks up the new chain.

- [ ] **Step 2: Auto-switch section** — after "The daemon log reads `clauth: codex auto-switched to '<name>' — live at the next codex session`", add:

> With `follow_active = true` the same switch repoints your `~/.codex/auth.json` and the log says so (`… — /home/you/.codex/auth.json follows it`). A wrap-off that clears the active slot leaves the link where it is, so your own codex keeps a working login.

- [ ] **Step 3: Configuration table** — add the row:

`| `follow_active` | bool | `false` | a codex switch also repoints the operator's `~/.codex/auth.json` link onto the new active profile's store ([Codex](Codex#switch)); only an already-adopted link moves |`

- [ ] **Step 4: Commit**

```bash
git add wiki/Codex.md wiki/Configuration.md
git commit -m "docs(codex): document follow_active in the codex and configuration pages"
```

### Task 6: the whole gate

- [ ] **Step 1: Run the whole suite** — `cargo test 2>&1 | tail -3`; expected `3908 passed; 2 failed; 1 ignored` (baseline 3902 + 6 new; the same two umask failures), or `3910 passed; 0 failed` under `umask 022`.
- [ ] **Step 2: Cross-model review** (gpt-6-astra, `--effort high`, read-only) of `git diff mommy..HEAD`; fix confirmed findings red-before-green; re-run the suite once more.
- [ ] **Step 3: Push** `feat/codex-follow-active` to the fork. Do not open the PR until the plan issue is up (decision D4).
