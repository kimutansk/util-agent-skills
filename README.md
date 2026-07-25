# util-agent-skills

General-purpose [Agent Skills](https://agentskills.io) for Claude Code. No
project-specific content — these are meant to be usable in any repository.

## Layout

```
manual/   skills with disable-model-invocation: true — run only when you type /name
auto/     skills Claude may load on its own
```

The split is for humans. Claude Code resolves a skill by the directory it is
linked into, not by where it lives here, so the grouping has no effect on
behaviour. It exists so that "does this run without me asking?" is answerable at
a glance — which matters when reviewing a change.

## Install

Skills are picked up from `~/.claude/skills/<name>/SKILL.md`. Claude Code follows
symlinks, so clone once and link the skills you want:

```sh
git clone https://github.com/kimutansk/util-agent-skills.git ~/src/util-agent-skills
ln -s ~/src/util-agent-skills/manual/claude-md-audit ~/.claude/skills/claude-md-audit
```

Link only what you use. Updating is `git pull`; Claude Code picks up SKILL.md
changes within a running session.

For a project-scoped install, link into that repository's `.claude/skills/`
instead. Note that `~/.claude/skills/` is not read by Cowork or cloud sessions —
those need the skill committed to the repository being worked on.

## Skills

### `manual/claude-md-audit`

Triage pass over a `CLAUDE.md`, rules file, or `SKILL.md`. Splits it into
individual rules and sorts them into three buckets: conventions worth testing for
redundancy, local facts that cannot be inferred and should stay, and guards
against irreversible outcomes that stay regardless.

The intent is to make an ablation pass affordable by shrinking what needs
testing. Read-only: it reads and reports, and does not modify the file.

## Conventions for skills in this repo

- Declare the narrowest `allowed-tools` that works. A skill can grant itself
  broad tool access, so anyone reviewing this repository should be able to see
  the blast radius from the frontmatter alone.
- Anything that changes state on disk or over the network gets
  `disable-model-invocation: true` and lives under `manual/`.
- Keep `SKILL.md` short and move detail into `reference/`, which loads only when
  needed.
- Findings that depend on a specific Claude Code version or model should say so
  in their output. Guidance for these tools shifts as the harness changes, and an
  undated conclusion outlives its accuracy.

## License

MIT
