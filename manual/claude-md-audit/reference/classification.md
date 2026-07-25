# Classification edge cases

Read this when a rule does not fall cleanly into convention / local fact / guard.

## Compound rules

A single bullet often carries more than one instruction:

> Run the test suite before committing, and never force-push to main.

The first clause is a convention; the second is a guard. Split it and classify
each part, keeping the same line reference for both. Report the split explicitly
— the user may want to break the bullet apart in the source file too, since a
compound rule is harder for a model to partially satisfy than two separate ones.

## Negative rules

"Never do X" needs care, because the class depends entirely on what X costs.

- *Never commit secrets* — guard. Irreversible once pushed.
- *Never use `any` in TypeScript* — convention. A linter would catch it and the
  fix is a diff.
- *Never write multi-line comment blocks* — convention, and a strong candidate
  for testing. Rules of this shape were often written against an older model's
  defaults; the newer guidance is to match the surrounding code's comment density
  rather than to prohibit outright.

The last case is worth calling out in Observations whenever it appears. A
prohibition that was calibrated against a previous generation's failure mode is
the archetype of a rule that now constrains without protecting.

## Scoped rules

Some rules only apply to part of the tree ("in `packages/api/`, handlers return
a Result type"). Classify by content as usual, but flag the scope: a rule that
applies to one directory is a candidate for the `paths` frontmatter field on a
skill, or for a nested `.claude/skills/` directory, rather than for the top-level
instruction file where it loads unconditionally.

## Conditional rules

"If the build fails, check the lockfile first" only fires in a state a probe has
to construct. These are Class A but expensive to test, because the probe must set
up the precondition. Mark testability as *conditional* and describe the state the
probe needs, so Phase 2 can decide whether the setup is worth it.

## Rules that restate a default

"Write working code." "Follow the existing code style." These are conventions
whose ablation result is easy to predict — but predicting is not measuring, and
the point of the exercise is to stop reasoning from assumption. Classify as A,
mark testability, and note in Observations that the expected result is *no
change*. If Phase 2 later shows an effect, that is a genuinely interesting
finding about the harness.

## Rules about the instruction file itself

"Keep this file under 200 lines." "Add new conventions to the bottom." These
govern maintenance rather than agent behaviour, and ablation says nothing useful
about them. Report them under non-directive content with a note, not as rules.

## Meta-instructions aimed at the user

Occasionally an instruction file contains notes written for humans — onboarding
steps, links to a wiki, a changelog. Not rules. Report the line ranges and their
token weight; moving them to a README is usually the right call, and it is often
the single largest reduction available in the file.

## When the rule is a fact that has gone stale

If a Class B rule describes something contradicted by the repository as it is now
— a path that no longer exists, a service that was retired — say so directly with
the evidence. This is the highest-value finding the audit produces, because a
confidently stated false fact does more damage than a redundant convention, and
no amount of ablation will surface it.
