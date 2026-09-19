# Morning report — codex parity, overnight 2026-09-19

Written while you slept. Read the first two sections before anything else.

## The one thing that changed everything

**A running codex will not switch accounts. And repointing the login link can corrupt a token store.**

That is Astra's verdict on the plan you left. It said `DISAGREE`, with codex's own source as
evidence. So **nothing was built.** No code, no install, no pull request, no issue. The night went
to spec work instead.

In plainer words:

- codex checks, every time it reloads its login file, that the account in the file is the same
  account it already had. A different account fails that check and codex returns an error. It does
  not adopt the new account.
- codex saves refreshed tokens **by file path**. If the link moves while a refresh is in the air,
  account A's new tokens can land in account B's file, and account A is left holding a token that
  is already spent.

Full text: `docs/superpowers/astra-spec-review-verdict.md`.

## The second thing you need to know

**I wrote a new design, had it reviewed, and the review found a way it could destroy a codex
login. I fixed it before you woke up.**

The new design, A2, is a small program named `codex` on your `PATH` that starts the active
account. My first draft forwarded **every** argument to it. That meant typing `codex login` would
have logged in **against your active managed profile** and replaced that profile's saved login —
and codex's login revokes what it finds on the server, with no way back. The wiki already warns
about exactly this; I re-created it by accident.

**The fix:** the shim only routes the commands that start a session — nothing, `exec`, `resume`,
`fork`, `review`. Everything else, and `login` and `logout` above all, goes straight to the real
codex and clauth does not touch it.

The review found eleven other things as well. I opened the code for every one of them, confirmed
each against the source, and fixed each. Both reviews are committed:
`docs/superpowers/astra-spec-review-verdict.md` and `docs/superpowers/astra-fold-verification.md`.

**These fixes have not been reviewed.** I stopped at two rounds on purpose (D19).

## Decisions I made without you

Each one is reversible, and each undo is written out in
`docs/decisions/2026-09-19-overnight-decisions.md`.

| # | What I decided | One-line undo |
|---|---|---|
| D11 | Slice A is **not** built. Astra falsified it. | Nothing to undo — nothing was built |
| D12 | The night becomes spec work: fold the review, draft **A2** | Discard the A2 section |
| D13 | **The handoff's premise was wrong, and I corrected it.** It said clauth already ships a `PATH` shim directory for claude at `~/.local/share/clauth/current@claude/bin`. It does not. That is the plugin install tree (`src/plugin_host.rs:442-447`), it has no `bin/`, and it is on no `PATH`. A2 creates a new `<data_dir>/clauth/bin` instead | Rewrite A2 step 2 |
| D14 | A2's three hazard answers: the recursion guard, falling through when no profile is active, refusing on Windows | Change the R3 / R3b / R3c answers |
| D15 | Slice B offers to split rename and disable/enable into their own PR | Delete R4b |
| D16 | The fold gets a **repair-verification** review, not a fresh discovery review | Ignore the verification report |
| D17 | The doc's live URL is stated as the **convention** URL and labelled unverified | Delete the `live` line |
| D18 | The second review said **REJECT**. I repaired A2 instead of shipping it, and confirmed all 12 findings against source first | Revert the commits on the branch |
| D19 | **No third review round.** The repairs are unreviewed, and the spec says so in its own text | Ask for another pass any time |

## What A2 is, in one paragraph

Typing `codex` should start the account clauth says is active. A2 does that with a small generated
program named `codex`, early on your `PATH`, that runs `clauth start <active-profile> -- "$@"` —
but **only for the commands that start a session**. `login`, `logout` and anything else go straight
to the real codex, untouched. Nothing is repointed. No credential file is written. A switch lands at
**the next `codex` launch**, never in the middle of a running session, because a running codex pins
which account it spends when it starts.

## Where the files are

- Spec: `docs/codex-parity-plan.md` on branch `docs/codex-parity-plan`, fork `crandrosoff/clauth`
- Fork link: https://github.com/crandrosoff/clauth/blob/docs/codex-parity-plan/docs/codex-parity-plan.md
- Rendered page: `docs/codex-parity-plan.html`, committed beside it
- Doc-harness URL (convention, **unverified** — see D17):
  https://2026-09-19-clauth-codex-parity-plan.3dstories.ca
- Issue text, still **held**, not filed: `docs/codex-parity-issue-draft.md`
- Astra's first verdict (DISAGREE, on the withdrawn slice A):
  `docs/superpowers/astra-spec-review-verdict.md`
- Astra's second verdict (REJECT, on A2 — now repaired):
  `docs/superpowers/astra-fold-verification.md`
- The withdrawn plan, marked withdrawn: `docs/superpowers/plans/2026-09-19-codex-follow-active.md`

**PR #87** (herdr plugin tags codex panes) is still open upstream and needs nothing from you.

## The one decision that is yours

**Is A2 — a `codex` shim on your `PATH` — the route you want?**

It is the only design I found that answers "typing `codex` should use the active account" without
writing a credential file. If you would rather have something else, nothing is built and nothing is
filed, so the cost of changing course is zero.

## What is NOT done

- Nothing is implemented. Not one line of Rust.
- The issue is **not** filed upstream. It is held for your read, per D4.
- No poll fixture was captured, so the claim "the empty `5h` cell is correct data" is **not** proven.
  The spec now says that out loud instead of asserting it.
- The round-two repairs are **unreviewed**. Two review rounds ran, not three (D19).
- I did **not** verify the doc-harness URL serves the page. The harness sits behind Cloudflare
  Access, and a made-up subdomain returns the same login page, so a 200 proves nothing (D17). You
  can check it in one click; I could not from here.
