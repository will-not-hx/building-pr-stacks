# building-pr-stacks

An agent skill for splitting a large change into a chain of stacked pull requests a reviewer
can actually read — one layer at a time, each one green on its own. Compatible with
[Claude Code](https://code.claude.com/docs/en/skills) and Codex.

## Install

### Claude Code

```sh
git clone https://github.com/will-not-hx/building-pr-stacks.git \
  ~/.claude/skills/building-pr-stacks
```

### Codex

```sh
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone https://github.com/will-not-hx/building-pr-stacks.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/building-pr-stacks"
```

Both tools discover the skill from the description in `SKILL.md` and can invoke it when work is
too large for one PR, when a reviewer has asked for something smaller, or when a stacked PR's
base has moved because a layer below it merged or took review changes. In Codex, you can also
invoke it explicitly with `$building-pr-stacks`.

## What it does

Three separate things make a stack reviewable, and only one of them comes naturally.

The first is the split. Given a large diff, a capable agent will already find reasonable seams —
domain types, then the client, then the mapper, then the wiring, then the endpoint — and will
work out on its own that the CHANGELOG belongs in the top PR so the layers don't all conflict on
the same lines.

The second is the reason each layer exists. A preparatory PR can be valuable because it isolates a
mechanical refactor, but *"in preparation for later work"* is not enough: its description must
name the immediate capability, the current constraint, the duplication or coupling it prevents,
and why reviewing it separately helps. If that case cannot be made, the layer belongs with its
consumer.

The third is the **furniture**: the thing that tells a reviewer where they are in the chain.
That is what goes missing. The skill supplies it as a required, copy-paste block — a `# | PR |
Base` table with a 👉 on the PR being read, "merge bottom-up", and an explicit claim that every
branch passes its suites alone — plus the naming scheme that makes position visible in a PR
list (`ABC-123 (3/5): …`) and keeps branch numbers agreeing with PR numbers.

It also covers the part that actually hurts: restacking. What to run depends on what happened,
and the four cases have genuinely different answers — a layer merging in a merge-commit repo
needs no rebase at all, while the same event in a squash repo rewrites the parent's commits and
forces a `git rebase --onto`. A review change at the *bottom* of an open six-PR chain is the
expensive one, and `git rebase --update-refs` moves every intermediate branch ref in a single
pass.

## How it was tested

Following Anthropic's `writing-skills` guidance, which adapts TDD to documentation: write the
failing test first, in the form of a scenario run against an agent that does *not* have the
skill.

The baseline scenario was a finished 21-file, +1,900-line branch and a reviewer refusing to read
it. The agent split it sensibly — and produced no stack table, no `(n/N)` titles, branch numbers
contradicting its own PR numbers, a parallel PR hanging off the default branch mid-chain, and
restack advice that prescribed a rebase the repo did not need. The omission was total and it was
structural, which set the form: `writing-skills` notes that a *prohibition* is the wrong shape
for a missing-element failure, and a required template is the right one.

Re-running the same scenario with the skill closed every one of those gaps. Two rounds of
critique then found real defects in it, both since fixed:

- A rule stating that an interface and its implementers must land together, sitting a few lines
  above a worked example that splits them across two PRs. Reading the real PR settled it: the
  port method lands **optional**, with a comment naming the layer that promotes it to required.
  The skill now documents that technique instead of the rule that contradicted it.
- A genuine impossibility. The stack table was required in every body, and PR 1's table has to
  cite PR 6 — which does not exist when PR 1 is posted. It now lands in two explicit passes.

## Licence

MIT — see [LICENSE](./LICENSE).
