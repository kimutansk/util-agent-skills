---
name: claude-md-audit
description: Classify every rule in a CLAUDE.md (or a rules/skill file) by whether it is worth testing for redundancy, and report which rules are safe deletion candidates. Read-only.
disable-model-invocation: true
argument-hint: [path-to-file]
allowed-tools: Read Grep Glob
---

# CLAUDE.md audit (Phase 1: triage)

## Environment

- Claude Code version: !`claude --version 2>/dev/null || echo "unknown"`

## What this does

Agent instruction files accumulate rules faster than they shed them. Some of those
rules genuinely change behaviour; others were written to work around a limitation
that the model or the harness has since outgrown, and now cost tokens while
competing for attention with the rules that matter.

The only reliable way to tell the two apart is ablation: remove a rule, run the
same prompt in a fresh session, and see whether the result changes. That is
expensive, and its conclusions expire whenever the harness or the model updates.

This skill is the step *before* ablation. It reads a target file, splits it into
individual rules, and sorts them into three buckets — so that ablation is only
ever spent on the rules where the answer is genuinely unknown. In practice this
removes most of the file from the test set.

**This skill does not edit anything.** It reads and reports. Deciding what to
delete stays with the user.

## What this deliberately does not do

Do not attempt to compare rules against Claude Code's built-in system prompt or
tool descriptions by introspection. Models cannot reliably quote their own system
prompt, so any "this duplicates the built-in instruction X" claim produced that
way is unfalsifiable and will read as more authoritative than it is. Behavioural
evidence is the only admissible evidence, and gathering it is Phase 2's job.

## Procedure

### 1. Resolve the target

Use `$ARGUMENTS` if given. Otherwise look for, in order: `./CLAUDE.md`,
`./.claude/rules/*.md`, `./.claude/skills/*/SKILL.md`. If several candidates
exist and none was named, list them and ask which to audit rather than guessing.

### 2. Split into rules

A "rule" is one instruction that could be independently removed. Usually a bullet
or a sentence, occasionally a short paragraph that only makes sense whole. Keep a
line reference for each so the report is actionable.

Section headers, prose that describes the repository, and code fences that exist
as illustration are not rules. Note them separately under "non-directive content"
— they still cost tokens, and a file that is 60% description is worth flagging
even though none of it is testable.

### 3. Classify each rule

Assign exactly one class. When a rule seems to fit two, the tie-break order is
**C beats B beats A** — misclassifying a safety rule as a style rule is the only
error here with real consequences.

| Class | Name | Meaning | Ablation? |
|---|---|---|---|
| A | Convention | Style, formatting, or process preference | **Test** |
| B | Local fact | Information the model has no way to know | Skip — keep |
| C | Guard | Prevents an outcome that cannot be undone | Skip — keep |

**Class B — local fact.** Ask: could a competent engineer derive this from the
repository alone? Filenames, internal invariants, "the staging bucket is X",
"table Y is append-only because of downstream Z" — none of this is inferable, so
ablation will always show an effect. Testing it wastes runs to confirm the
obvious. Keep it, and instead check whether it is still *true*: stale local facts
are worse than absent ones, because they are stated with confidence.

**Class C — guard.** Ask: if this rule were ignored, could the result be reverted
with `git revert` or an equivalent? If no — data loss, a published artifact, an
external side effect, a credential in a commit — it is a guard. Guards stay
regardless of what ablation would show, because the cost of the rare failure
dominates the token saving. Note that the source article's advice to avoid
over-constraining is explicitly qualified as applying outside highly important
areas; this bucket is that exception.

**Class A — convention.** Everything else. These are the rules worth testing,
because a model or harness update may have made them redundant without anyone
noticing.

Read `reference/classification.md` for edge cases: rules that bundle several
instructions, negative rules ("never do X"), rules that only apply to part of the
tree, and rules whose effect is conditional on other rules.

### 4. Assess the Class A rules

For each Class A rule, record two things the user will need later:

- **Testability** — can its effect be observed in the output of a single prompt?
  A rule about commit message format is observable. "Think carefully before
  refactoring" is not, and should be flagged as untestable rather than being
  handed to Phase 2 as if it were.
- **Probe sketch** — one sentence describing the task that would expose the rule
  if it is doing work. Probe design is the real bottleneck in Phase 2, so
  capturing the idea here, while the rule's intent is in view, saves effort later.

### 5. Report

Use this structure:

```markdown
# CLAUDE.md audit — <path>

Audited against Claude Code <version>, model <model>.
Findings are specific to this combination and should be re-run after either changes.

## Summary
<N> rules: <a> convention (testable), <b> local fact, <c> guard.
<M> lines of non-directive content.

## Class A — convention (ablation candidates)
| Line | Rule | Testable | Probe sketch |

## Class B — local fact (keep; verify accuracy)
| Line | Rule | Still accurate? |

## Class C — guard (keep unconditionally)
| Line | Rule | Irreversible outcome prevented |

## Non-directive content
<line ranges, with a note on token weight>

## Observations
<Anything structural: rules that contradict each other, rules restating a
default, sections that would be better as a separate skill loaded on demand.>
```

Stamping the version and model matters more than it looks. A finding of "this
rule appears redundant" is a statement about one harness build, and an undated
report will be reused long after it stopped being true.

Present the report in the conversation. Write it to a file only if the user asks,
and say which path you would use before doing it.

## Calibration

Err toward Class B and C. A convention rule wrongly parked in B costs a few
tokens; a guard wrongly parked in A invites its deletion.

If a rule's purpose is unclear, say so in the Observations section rather than
forcing a class. "I could not tell what this rule is for" is a useful finding on
its own — it usually means the rule outlived the situation that produced it.
