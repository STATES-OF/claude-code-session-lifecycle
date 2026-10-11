# STATES-OF/claude-code-session-lifecycle

Models a Claude Code session's own real internal state (layer 1, per
`states-of`'s README) around context-window exhaustion and recovery.
Built from: Anthropic's documented behavior (context-window / troubleshooting
docs), community bug reports (unverified, flagged below), and one real
observed instance (`observations/2026-10-10-ao-pfm.md`).

Layer-1 facts here are only as good as the documentation and the one real
instance available. Anything marked `unverified` is a plausible mechanism,
not a confirmed one — don't let a future reader of this repo treat it as
settled just because it's written down.

## States

| State | Meaning |
|---|---|
| `ACTIVE` | Normal turn-taking, context has headroom |
| `NEAR_LIMIT` | Context above the auto-compact trigger threshold (default behavior per docs; threshold adjustable down only, via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`) |
| `AUTO_COMPACTING` | Claude Code is attempting its own automatic compaction |
| `CONTEXT_EXHAUSTED` | Hard stop: context full, no further turn can proceed without `/compact` or `/clear`. Surfaces literally as `Context limit reached · /compact or /clear to continue`. |
| `COMPACT_THRASHING` | A documented failure sub-state of `CONTEXT_EXHAUSTED`: repeated auto-compact attempts fail to free enough space, and/or a manual `/compact` fails outright (seen in the wild as "Not enough messages to compact" or an explicit thrashing error) |
| `EXITED` | Process has actually terminated (real OS-level fact, layer 1 — see the observation-layer repo for how this is distinguished from an intermediary merely *showing* an exited-looking screen) |

## Transitions

```
ACTIVE --(context grows)--> NEAR_LIMIT
NEAR_LIMIT --(auto-compact fires, succeeds)--> ACTIVE          [documented default behavior]
NEAR_LIMIT --(auto-compact fires, fails/thrashes)--> COMPACT_THRASHING   [unverified mechanism; see below]
NEAR_LIMIT --(context fills before auto-compact fires)--> CONTEXT_EXHAUSTED
CONTEXT_EXHAUSTED --(user/agent sends /compact)--> ACTIVE        [if it succeeds]
CONTEXT_EXHAUSTED --(user/agent sends /compact)--> COMPACT_THRASHING  [if it fails]
CONTEXT_EXHAUSTED --(user/agent sends /clear)--> ACTIVE          [always succeeds; full history loss]
COMPACT_THRASHING --(/clear)--> ACTIVE                            [the guaranteed escape hatch]
ANY --(/exit, or process killed)--> EXITED
```

## The known "wall" and its fallback

`/compact` is not guaranteed to produce a transition back to `ACTIVE`.
Two real documented failure shapes exist (Anthropic's own troubleshooting
docs): an explicit "Autocompact is thrashing" error, and a `/compact`
call that itself errors with "Not enough messages to compact." Both are
represented here as `COMPACT_THRASHING` rather than a successful
transition — i.e. attempting the `/compact` path can genuinely hit a wall
with *no* transition to the state you wanted (`ACTIVE`, fully compacted).

**The fallback that always works is `/clear`.** It has no documented
failure mode — it is a full reset, not a negotiation with existing
content, so there is nothing for it to fail to compact. The cost is total
in-context history loss. Whether that cost is acceptable depends entirely
on whether the agent's real continuity lives in-context or in durable
external state (status files, chronicle logs) it re-reads on
`SessionStart` — a design property of the *agent*, not of Claude Code
itself. See `observations/2026-10-10-ao-pfm.md` for a real case where this
matters.

## Open / unverified

- Whether `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` / `CLAUDE_CODE_AUTO_COMPACT_WINDOW`
  being unset, vs. actively misconfigured, vs. a version-specific bug
  (one community report claims auto-compact is "silently disabled for
  non-remote sessions" below v2.1.159 unless one of those is set) is what
  actually causes `NEAR_LIMIT` to skip straight to `CONTEXT_EXHAUSTED`
  instead of auto-compacting. **Not confirmed against this machine's
  actual Claude Code version.**
- Whether a session relaunched into the same pty via `--continue`/`--resume`
  while already `CONTEXT_EXHAUSTED` reloads the full bloated transcript
  (re-landing in `CONTEXT_EXHAUSTED` immediately) vs. starting genuinely
  fresh. The one real observation on file is consistent with the former
  but doesn't prove it (see observations log).
