---
name: building-pr-stacks
description: Use when work is too large for one pull request, when a reviewer has asked for smaller PRs, before opening more than one PR for the same ticket, when splitting a finished branch into reviewable layers, or when a stacked PR's base has moved because a layer below it merged or took review changes.
---

# Building PR Stacks

## Overview

A stack is one linear chain of PRs, each based on the one below it, each **independently green**. The reviewer holds one layer at a time and can approve it without trusting the layers above.

Three things make a stack reviewable: the split, each layer's reason to exist, and the furniture that tells a reviewer where they are in the chain. Agents reliably supply the split; they can still leave a preparatory PR unexplained even when its Stack table is perfect. **The Stack block below is required in every body.**

## When to use

- One ticket whose change crosses layers (types → client → mapper → wiring → endpoint)
- A finished branch a reviewer has refused as too large
- A ticket with subtasks that each stand alone

Not for: a change with one seam (open one PR), or unrelated changes (open separate PRs — a stack implies dependency).

## 1. Find the seams

One PR per layer, dependencies pointing downward, so no layer needs the ones above it to compile or pass.

### Preserve quality when splitting

Splitting PRs changes review size, not the quality bar. Every layer must preserve appropriate code ownership, repository conventions, validation, error handling, tests and release checks. Passing CI alone does not make a split sound.

Do not put a server-owned schema in a client folder, duplicate a contract, weaken tests or move files outside a version gate just to make a preparatory PR independently green. Keep related ownership and dependency changes together. If a seam requires a quality compromise, move the seam or combine the PRs; a later cleanup PR is not justification for introducing it.

Check both each intermediate layer and the completed stack. Preparation can be unused until its consumer lands, but it must still be maintainable in its own right.

| Rule | Why |
|---|---|
| A tooling or lint gate lands **first**, carrying the file that trips the rule | Landing it last re-churns every layer below it; separating override from offending file leaves one of the two branches red |
| A spike that proves an upstream assumption goes **below** what relies on it | The reviewer sees the evidence before the code betting on it |
| A preparatory or no-public-behaviour layer names its immediate consumer in the PR body | Independently green makes the layer safe, not self-explanatory. State the capability it enables, the current constraint, the duplication, coupling or risk it prevents, and why separate review helps. If that case cannot be made, fold it into its consumer |
| The endpoint or entrypoint lands **last** | Layers below have no caller, so merging them changes no behaviour |
| An interface change lands **with its implementers**, or lands **optional** | A required port method with no implementer breaks compilation. `getState?:` plus a comment naming the layer that promotes it to required lets the contract land first — what ABC-123 (1/5) does |
| CHANGELOG and version bumps follow the repository's release rules and the layer that needs them | Usually the final endpoint layer; earlier public changes may need their own. Do not bypass a gate or misplace code to reserve the bump for the last PR |

Sanity check: roughly 50–550 production lines and under ~10 files per layer. Past that, look for the seam you missed.

Tests move with the code they cover, so a layer can be mostly tests. A test file spanning two layers is a seam signal — split the file, or move the seam.

**Keep it one chain.** Don't hang a parallel PR off the default branch mid-stack — the Base column stops describing a readable order.

## 2. Name the chain

Pick the scheme that matches how the work is ticketed. Branch numbers and PR numbers must agree.

| Situation | Title | Branches |
|---|---|---|
| One ticket, split by layer | `ABC-123 (3/5): Map provider state responses into the domain model` | `ABC-123`, `ABC-123-2-provider-client`, `ABC-123-3-provider-mapper`, … (first is bare) |
| One subtask per layer | `ABC-207: Add the record cleanup cron entrypoint` | `ABC-201`, `ABC-202`, `ABC-203`, … |
| No ticket per layer | `Draggable inspector column` | `ui-stack-01` … `ui-stack-12` |

Each branch is the base of the next.

## 3. The Stack block (REQUIRED in every body)

Bottom of the body, after a `---`. Fill every slot; the 👉 marks the PR being read.

The table cites numbers that don't exist until the chain is open, so it lands in **two passes**: post the bodies without a Stack section at all, then one `gh pr edit --body-file` per PR adding the finished table. The second pass is part of opening the stack. Never post a table of placeholder numbers. In that second pass, also replace prose such as *"the next layer"* with a link to the real consuming PR, especially in preparatory layers.

```markdown
---

### Stack

Part of [ABC-123](https://jira.example.com/browse/ABC-123), split one PR per layer. **Merge bottom-up.**

| # | PR | Base |
|---|---|---|
| 1 | [ABC-123 (1/5) Domain types + port method](https://github.com/OWNER/REPO/pull/101) | `staging` |
| 2 | [ABC-123 (2/5) Provider client `fetchState`](https://github.com/OWNER/REPO/pull/102) | `ABC-123` |
| 3 | 👉 [ABC-123 (3/5) Provider → domain mapper](https://github.com/OWNER/REPO/pull/103) | `ABC-123-2-provider-client` |
| 4 | [ABC-123 (4/5) Service + telemetry wiring](https://github.com/OWNER/REPO/pull/104) | `ABC-123-3-provider-mapper` |
| 5 | [ABC-123 (5/5) `GET` endpoint](https://github.com/OWNER/REPO/pull/105) | `ABC-123-4-service-wiring` |

Every branch in the stack passes `npm run build` and `npm test` on its own.
```

That last line is a claim you have to have earned on every branch. When one layer touches a directory with its own suite, keep the scopes separate rather than widening the claim:

> Every branch in the stack passes `npm run build` and `npm test` on its own; PRs 3 and 4 also pass `cd test-ui && npm test`.

Run them on each branch or delete the line.

**Long incremental stacks** (10+, opened as you go) use a one-line position lead at the *top* instead of the table, because the table goes stale faster than it helps:

> 5 of 11, and the load-bearing one — PRs 6 to 11 build on the interfaces it defines. Stacked on #101.

**Hoist a cross-PR blocker into a blockquote** on the PR that carries the risk, rather than burying it in prose:

> **This feature must not reach production until the paired rollback story lands.**

For everything else in the body, use the `writing-pr-descriptions` skill.

## 4. Build it

Splitting a branch that is already finished — keep it as the reference and restore file subsets from it:

```sh
git branch -m ABC-123 ABC-123-full          # never becomes a PR
git checkout -b ABC-123 staging
git restore --source=ABC-123-full -- src/domain/state.ts src/ports/provider.ts
git commit -am 'Add state domain types and port method (ABC-123)'
npm run build && npm test                    # green before the next layer
git push -u origin ABC-123
gh pr create --base staging --title 'ABC-123 (1/5): ...' --body-file body-1.md
```

Repeat per layer, each `git checkout -b <next> <previous>` and `gh pr create --base <previous>`.

Before opening any of them, prove the carve-up dropped nothing — the top of the stack must equal the reference:

```sh
git diff ABC-123-5-endpoint ABC-123-full     # must be empty
```

Open the whole chain at once so the reviewer sees the shape, then make the table pass above.

Don't push the `-full` reference branch, and delete it only once the top of the stack has merged.

## 5. Restack when a base moves

| What happened | Do this |
|---|---|
| A layer merged, repo uses **merge commits** | Nothing to rebase — GitHub retargets the child automatically. Refresh the tables. |
| A layer merged, repo allows **squash** | The parent's commits were rewritten, so the child now carries duplicates. `git rebase --onto staging <old-parent-branch> <child>` before anything else. |
| The **bottom** took review changes | Amend it, then from the top: `git checkout <top> && git rebase --update-refs <bottom>` — one rebase moves every intermediate branch ref. Then `git push --force-with-lease --atomic origin <b1> <b2> <b3>`. |
| A base branch is gone | `gh pr edit <n> --base <branch>` |

Then **refresh the Stack table in every body** — the Base column and the 👉 both go stale silently, and re-run the suites so the independently-green line stays true.

## Common mistakes

| Instead of | Write |
|---|---|
| `Based on #102` in prose | The Stack table, in every body |
| `Add the domain types (ABC-123)` | `ABC-123 (1/5): Add state domain types and port method` |
| Branch `ABC-123-4-route` on PR 5 | Branch number matches PR number |
| A parallel PR off `staging` mid-chain | One linear chain |
| "Extract shared helper for future work" | Name the immediate consuming layer, why the current code cannot support it cleanly, what duplication or coupling the refactor avoids, and why it is easier to review separately |
| "Every branch is green" (unrun) | Run build and tests on each branch, or drop the line |
| A body posted with `<n>` placeholders | Counts read off an actual run |
| Retargeting and rebasing after every merge by reflex | Check the repo's merge strategy first — a merge-commit repo needs neither |
