## Codex parity: a follow-up series, offered for rulings before any PR

#69 landed the codex harness: the second roster, capture and browser login, `clauth start <codex>`, the usage poll, a separate chain, and a read-only Overview section. By `docs/codex-plan.md`'s parity map, everything else a Claude Code account gets — row actions, a Setup page, the chain editor, the Usage breakdown, the Config keys, the Tokens lens, `delegate` — stops at the harness line. `wiki/Codex.md` says it plainly: "the shell verbs above are the whole surface".

I use clauth with two codex workspaces beside four Claude accounts and would like to carry codex over that line, one slice per PR, each mergeable on its own and each keeping #69's rulings. Before writing code past the first slice I would rather have your rulings than guess. The full spec, in the style of `codex-plan.md`, is here: **`docs/codex-parity-plan.md` on my fork** (link below). Summary:

**Pinned:** two rosters stay; tabs learn a second list through one `RowSel { Claude(i) | Codex(i) }` selection type, not a merged roster; the TUI calls the CLI's own codex writers; nothing reads a wire that does not exist (kick, spend ceiling, per-session fallback, keychain stay out).

**Slices, in order:**

- **A. `follow_active`** (opt-in, default off): a codex switch — manual or the chain's — also repoints the operator's `~/.codex/auth.json` link onto the new active profile's store, through the same tmp-and-rename the capture uses, only when that file is already a link into `~/.clauth/profiles/*/auth.json`. A plain `codex` then starts on the chain's pick; a running one adopts it at its next reload-before-refresh. One function (`switch_codex_profile`), both callers. ~60 lines + tests + two wiki paragraphs. *I have this one built on a branch and can open it first.*
- **B. Overview + Setup:** codex rows selectable; actions `switch`, `re-login`, `disable`/`enable` (new to codex, added to `CodexState` and the CLI first), `delete`, `rename`, reorder; a Setup page with only the rows a codex profile has, and a `+ codex login` row.
- **C. Fallback tab:** the codex chain editor (order, weekly line, `wrap_off`, `follow_active`) — ends the hand-edited file.
- **D. Usage tab:** codex rows with the windows the wire reports. Measured on codex 0.155 / a Pro plan: `rate_limits.primary.window_minutes = 10080`, `secondary = null` — one window. The 5h dash today is correct data; D renders it as `weekly only`, and the wiki's "two windows" sentence becomes "one or two".
- **E. Config tab:** the codex keys.
- **F. Tokens tab:** a codex cost lens fed from `~/.codex/sessions/**/rollout-*.jsonl` `event_msg/token_count` events, keyed by session, with the same estimate label the claude lens carries.
- **G. `delegate` over `codex exec`** with a per-harness result formatter, and the Plugin tab listing codex live sessions. `AppConfig.profiles` stays claude-only; the MCP layer resolves a codex name through `CodexState` explicitly.

**Rulings I would like before B onward:** R1/R2 the `follow_active` default and name; R3 on `Off`, leave the link (spec) or detach; R4 the codex action set; R6 one Fallback tab with two sections, or a `c` filter; R7 reading the rollout `token_count` feed as a usage supplement; R8/R9 the rollout files as the Tokens data source and pricing by `turn_context.model`; R10 which `codex exec` output contract to pin; R11 whether `delegate` may run a non-active codex profile.

Refs #45, #50, #87.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
