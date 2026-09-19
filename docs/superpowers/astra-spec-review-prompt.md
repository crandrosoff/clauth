# Bounded consult: review a spec and an issue draft before they go upstream

You are reviewing, read-only, in the clauth repository fork checkout at `/home/rocky00717/rawgentic/projects/clauth/.worktrees/parity-plan` (branch `docs/codex-parity-plan`, based on upstream `mommy` at 37b4ca08). Do not edit anything. Do not run the `clauth` binary. Read files and `git log`/`git diff` freely.

## What to read, in this order

1. `docs/codex-parity-plan.md` — the spec under review (134 lines).
2. `/tmp/clauth-issue-codex-parity.md` — the GitHub issue text that would carry the spec to the maintainer.
3. `docs/codex-plan.md` — the maintainer's own spec for the codex series that already merged (#69), with its parity map, settled questions and "deviations as shipped". The new spec must not contradict its rulings.
4. `wiki/Codex.md` — the shipped behavior the spec builds on. Read all of it.
5. Code anchors the spec cites: `src/actions.rs` (`switch_codex_profile` ~1143, `adopt_operator_auth_slot` ~1588, the capture around 1370-1420), `src/main.rs` (`cmd_switch` ~1683), `src/usage/scheduler.rs` (~3618-3648, the codex auto-switch), `src/codex_profiles.rs`, `src/tui/app.rs` (`HarnessFilter` ~1519, `codex_rows` ~1573, `ActionMenuState` ~833, the `c` key ~3196, `poll_codex_rows` ~10130), `src/tui/render/overview.rs` (74-122).

## What I need from you

Answer in this exact shape, printed as your final message in the terminal (you cannot write files), nothing else:

```
VERDICT: AGREE | AGREE-WITH-CHANGES | DISAGREE

## Slice A: the falsifying case
<The strongest concrete scenario in which `follow_active` — repointing `~/.codex/auth.json` onto another profile's store on a switch — loses or double-spends a single-use refresh token, strands a running codex, or fights clauth's own refresher. Cite the code or wiki passage that makes it real, or state plainly that you could not construct one and why.>

## Material objections (ranked, most severe first)
<Each: what is wrong, where (file:line or spec section), why it matters, what to change. Only things that would change the design or the maintainer's answer. Skip style.>

## Contradictions with codex-plan.md rulings
<Any place the spec reverses a settled question, a pinned decision, or a deviation-as-shipped. Quote both sides.>

## Issue text
<Is it accurate against the spec and the code? Anything the maintainer will push back on immediately? Wording that overclaims?>

## Copy fixes
<Short list, optional.>
```

Be adversarial. A review that agrees with everything is worth nothing to me; find the case that breaks slice A first, then the rest.
