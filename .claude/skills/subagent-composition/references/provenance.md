# Provenance — how the mechanics in `SKILL.md` were established

Moved out of the skill body on 2026-08-01. Nothing was cut: the body carried this
as narration, and narration in a skill body competes with the procedure for the
surviving prefix while telling a reader nothing they must *do*. It is evidence,
so it belongs where evidence is kept.

## What was checked, and how

**Documentation pass — July 19, 2026.** The mechanics were verified against the
Claude Code documentation: `sub-agents`, `agent-sdk/subagents`, `worktrees`,
`settings`, `model-config`, and `plugins-reference`.

**Live probe — CLI 2.1.214.** Five throwaway agents differing by exactly one
field, with canary tokens planted in a skill body and in a `CLAUDE.md`, run
against a no-`skills` control.

## What the control bought

The control earned its place twice, and both are the reason the inheritance
ledger in the body can be stated as fact rather than as belief:

1. It distinguished a **hot-reload failure** from **bad YAML** — two causes with
   identical symptoms, which without a control would have been read as whichever
   the prober expected.
2. A **positive control** distinguished a **real negative** from a **blind
   probe**. A probe that cannot demonstrate it would have detected the thing is
   not evidence of the thing's absence.

That second one is the positive-control duty, and it is why three behaviours in
the body are stated as "cost a probe to find" rather than inferred from docs:
agent registration lagging the filesystem in both directions, `--agent {name}`
silently ignoring `skills:`, and `tools:` replacing inheritance rather than
trimming it.

## Design lineage

Composition discipline here is the Lexideck toolkit applied to the agent
boundary: the frames are hypershots, the block order is attention management, and
the return contract is decoherence prevention at the one place where ambiguity is
unrecoverable — a sub-agent's final message, which has no second chance to be
corrected.

## What is still unmeasured

The probes established the **mechanics** — what crosses the boundary, what the
frontmatter fields do, what the ephemeral call cannot express. They did not
measure whether a prompt composed by these frames produces better sub-agent
output than one that is not. That is a separate question and this repository has
no evidence for it. `probes/` holds the diagnostics; there is no `VALIDATION.md`
here yet, and until there is, treat the composition guidance as engineering
judgment and the mechanics as verified.
