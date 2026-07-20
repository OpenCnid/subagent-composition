# Findings: the Claude Code sub-agent boundary

**Probed:** 2026-07-19 · **Claude Code:** 2.1.214 · **Platform:** Windows 10, PowerShell + Git Bash
**Reproduce:** `bash probes/run-probes.sh`

Every row below is either cited to the Anthropic docs or observed directly by
the probe harness. Nothing here is written from a model's prior. Where the two
sources disagree, both are recorded.

Undocumented behavior carries no compatibility promise. Re-run the harness
before relying on any ⚠️ row.

---

## 1. What crosses the boundary

The load-bearing fact about sub-agents: they inherit far less than intuition
suggests.

| Crosses | Does **not** cross |
|---|---|
| The agent's own prompt (file body, or the `prompt` param) | The parent's conversation history and every tool result in it |
| Project `CLAUDE.md` (subject to `settingSources`) | The parent's system prompt and harness instructions |
| Tool *definitions* for its allowlist | Auto-memory / `MEMORY.md`, including recalled entries |
| Skills named in `skills:` — **full body** | Skill content the *parent* has loaded |
| Session extended-thinking config | The parent's reasoning, rejected paths, and the user's stated intent |
| Names of sibling agents (for `SendMessage`) | — |

**Return channel: the final message only.** Intermediate tool calls and their
results stay in the sub-agent's context and are discarded unread. A sub-agent
that works perfectly and signs off with "Done — see above" has produced nothing.

`CLAUDE.md` inheritance was verified directly: a probe run in a directory whose
`CLAUDE.md` contained the token `MARMOSET-7731` reported that token back.

**Exception — forks.** `/subtask` and `--fork-session` inherit full conversation
context, system prompt, and tools. When continuity matters more than isolation,
fork instead of spawning. To continue an existing agent rather than cold-start a
new one, use `SendMessage` with its name or `agentId`.

---

## 2. Frontmatter reference

`.claude/agents/{name}.md`, YAML frontmatter followed by the body (which is the
agent's system prompt).

| Field | Value | Notes |
|---|---|---|
| `name` | kebab-case | Required. Precedence: project `.claude/agents/` > user `~/.claude/agents/` > plugin. |
| `description` | plain scalar | Required. Drives automatic delegation. Quote it if it contains `: ` (colon-space), which YAML parses as a mapping. |
| `tools` | **comma-separated string** — `Read, Grep, Glob` | ✅ verified. Not a YAML list. Omitted ⇒ inherits everything. MCP form `mcp__server__tool`. |
| `disallowedTools` | comma-separated string | Applied *before* `tools`. Supports `mcp__server__*` and `mcp__*`. Not independently probed. |
| `model` | `inherit` \| `opus` \| `sonnet` \| `haiku` \| `fable` \| full ID | Default `inherit`. `CLAUDE_CODE_SUBAGENT_MODEL` sets a floor. |
| `effort` | `low`…`max` or number | Per-agent reasoning budget. Valid in filesystem frontmatter. |
| `maxTurns` | integer | Hard ceiling. The cheapest guard against an unbounded sweep. |
| `skills` | any YAML form | ✅ verified — see §3. Loads the full skill body. |
| `isolation` | `worktree` | Own git worktree; auto-removed **only if unchanged**. Not independently probed. |
| `color`, `hooks` | — | Filesystem-only, undocumented shape. `hooks` / `mcpServers` / `permissionMode` are forbidden in plugin-shipped agents. |

**Recursion.** A sub-agent can spawn its own only if `Agent` is in its `tools`;
depth caps around five levels.

---

## 3. `skills:` accepts every YAML form ✅

All three spellings are equivalent and all three load the complete skill body:

```yaml
skills: hypershot-protocol          # bare scalar / comma-separated
skills: [hypershot-protocol]        # flow sequence
skills:                             # block sequence
  - hypershot-protocol
```

**Evidence.** Three agents identical but for this field, plus a control with the
field absent. Each was asked for two tokens it could only hold if the skill body
were present: a distinctive proper noun from deep in the skill, and the verbatim
title of one of its numbered rules. Tools were restricted to `WebSearch` so no
agent could read the skill off disk.

| agent | `skills:` form | canary | rule title |
|---|---|---|---|
| `probe-skills-bare` | scalar | ✅ recovered | ✅ verbatim |
| `probe-skills-flow` | `[flow]` | ✅ recovered | ✅ verbatim |
| `probe-skills-block` | block seq | ✅ recovered | ✅ verbatim |
| `probe-skills-control` | *absent* | `absent` | `absent` |

`skills:` is the **only** way to give a sub-agent skill content. Skills loaded
in the parent session do not cross.

---

## 4. `tools:` is enforced, not advisory ✅

Every probe declared `tools: WebSearch` and every probe reported exactly one
available tool. The allowlist **replaces** inheritance rather than trimming it.

This also confirms the comma-separated syntax at runtime: had the field failed
to parse, the agents would have inherited the full tool set, and the probes
would have reported it.

---

## 5. ⚠️ `--agent {name}` silently ignores `skills:`

**The most expensive finding here.** Session-agent mode and sub-agent spawn are
different code paths, and only the spawn path preloads skills.

Same agent definition, same canary, two invocation modes:

```bash
# session-agent mode — skills: is dropped, no error, no warning
claude --agent probe-context-detect -p "..."
#   CLAUDEMD_CANARY: MARMOSET-7731
#   SKILL_CANARY: absent          ← the skill was declared and did not load

# true sub-agent spawn — skills: is honored
claude --allowedTools "Agent" -p "Spawn the probe-context-detect subagent via the Agent tool …"
#   CLAUDEMD_CANARY: MARMOSET-7731
#   SKILL_CANARY: Nexus Singularity Engine
```

Note the first result is *partially* correct — `CLAUDE.md` still crosses — which
is exactly what makes it convincing. The agent looks like it is receiving
context, so the missing skill reads as "`skills:` is broken" rather than "wrong
invocation mode."

**Practical rule:** validate agent behavior through an actual spawn. Never
through `--agent`. A working definition will otherwise read as broken, and the
failure is silent in both directions.

---

## 6. ⚠️ Agent registration lags the filesystem — both ways

Skills register the instant the file is written. Agents do not.

- A newly written agent file fails with **`Agent type 'x' not found`** while its
  definition is entirely valid.
- A **deleted** agent file can keep appearing in the available-agents list well
  after removal. Observed directly: five probe agents were announced as newly
  available *after* they had been deleted from disk.

Never read `Agent type not found` as evidence of a broken definition. To test a
new agent without waiting or restarting, use a fresh process:

```bash
claude --allowedTools "Agent" -p "Spawn the {name} subagent via the Agent tool, \
passing the prompt '{...}'. Then output its final message verbatim and nothing else."
```

---

## 7. Not probed

Stated for honesty; do not treat these as verified.

- `disallowedTools` ordering relative to `tools`.
- `isolation: worktree` cleanup semantics and the `cleanupPeriodDays` sweep.
- Behavior at the `maxTurns` ceiling — whether it truncates or returns partial output.
- Wildcard forms (`*`) inside `tools:`.
- Automatic-delegation matching: the docs state `description` drives it but
  publish no algorithm, and we did not attempt to characterize one.
- Whether any of this holds on non-Windows platforms or other CLI versions.

---

## Sources

- [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Worktrees](https://code.claude.com/docs/en/worktrees) ·
  [Settings](https://code.claude.com/docs/en/settings) ·
  [Model config](https://code.claude.com/docs/en/model-config) ·
  [Plugins reference](https://code.claude.com/docs/en/plugins-reference)
- Probe harness: [`probes/`](../probes/), method in
  [PROBE-METHODOLOGY.md](PROBE-METHODOLOGY.md)
