---
name: pr-description
description: Use when writing a pull request description, PR body, or PR title for a branch — including "write me a PR description", "create a PR according to the git workflow", or preparing a branch for review.
---

# PR Description

## Overview

A PR body exists so a reviewer can answer two questions without asking you: **is this change
correct** (the Why) and **how do I confirm it works** (the verification steps). Everything else is
already in the Files Changed tab.

Adapt to the repo's own convention when it has one. Fall back to the default structure below when it
doesn't.

## 1. Find the repo's convention

Check in this order and stop at the first hit:

```sh
ls .github/PULL_REQUEST_TEMPLATE.md .github/pull_request_template.md .github/PULL_REQUEST_TEMPLATE/ 2>/dev/null
ls CONTRIBUTING.md docs/*GIT* docs/*WORKFLOW* docs/*CONTRIB* 2>/dev/null
gh pr list --state merged --limit 5 --json title,body   # infer from what the team actually ships
```

Read the file you find — do not write from memory of how that repo "usually" does it. If it defines
a template, section headings, emoji, or label table, **that file wins over this skill**. Mirror its
headings verbatim, including emoji and ordering.

No convention found anywhere → use the default structure in section 4.

## 2. Find the real base branch

**Do not assume `main`/`master`, and do not trust `origin/HEAD`.** Repos with a staging flow point
`origin/HEAD` at the production branch while every feature PR actually targets the integration
branch. Integration branches seen in practice: `staging`, `development`, `develop`, `dev`, `main`,
`master`.

Determine it empirically:

```sh
gh pr list --state merged --limit 10 --json baseRefName --jq '.[].baseRefName' | sort | uniq -c
git branch -a | grep -Ei 'staging|develop|production|main$|master$'
```

The base is whatever recent feature PRs were merged into. Cross-check against the workflow doc from
step 1 if there is one. Then:

```sh
git log --oneline <base>..HEAD        # this branch's own commits
git diff <base>...HEAD --stat         # scope
git diff <base>...HEAD                # read all of it
```

If the local base branch is stale or absent, fall back to `git diff HEAD~<n>` covering exactly the
branch's commits — and say in your reply which reference you used.

## 3. Read the hunks

Read the actual diff, not the diffstat. A description written from `--stat` restates filenames and
gives the reviewer nothing. The Why section is impossible to write without having read the code.

Pull the root cause from the diff and from `git log` on the lines being changed
(`git log -S'<symbol>' --oneline`) when a fix reverses an earlier decision — naming the commit that
introduced the bug is often the most useful sentence in the PR.

## 4. Default structure

Use when the repo has no template of its own.

```md
## <emoji> <prefix>: <name>

### 🔧 What changes were made?
### ❓ Why were the changes necessary?
### 🛠️ How were the changes implemented?

## 🔗 Related Issues

## ✅ Steps to Test or Verify Changes
```

Section contract:

| Section | What goes in it |
| --- | --- |
| What | One framing sentence, then a bullet per user- or ops-visible change. Name the surfaces touched (API, admin UI, client app, schema, CI). |
| Why | The root cause or the requirement — the thing not visible in the diff. Symptom → mechanism → consequence. |
| How | Bullets naming real symbols and paths: the function, the config key, the migration directory, the model. |
| Related Issues | `- Relates to #<!-- ticket -->`. **Never invent an issue number.** Use one only if it appears in a commit message, branch name, or the user told you. |
| Steps to Test | Numbered steps a reviewer runs verbatim. |

**Why ≠ What.** "Added an index" is What. "The aggregation sequential-scanned a table that grows with
every page view" is Why. If Why paraphrases What, you skipped step 3.

### Title, emoji and label

Format: `<emoji> <prefix>: <short description>`

| Type | Prefix | Emoji | Label | Usage |
| --- | --- | --- | --- | --- |
| Feature | `feat` | 🚀 | `feature` | New functionality |
| Fix | `fix` | 🔧 | `fix` | Bug fixes |
| Hotfix | `hotfix` | 💉 | `hotfix` | Critical production fix |
| Refactor | `refactor` | 🧹 | `refactor` | Code restructuring, no behavior change |
| Chore | `chore` | 🐢 | `chore` | Maintenance, deps, tooling, CI |
| Config | `config` | ⚙️ | `config` | Configuration changes |
| Styles | `styles` | 🎨 | `styles` | UI / styling only |
| Tests | `tests` | 🧪 | `tests` | Tests only |
| Release | `release` | 🎈 | `release` | Integration → production merge |

Examples: `🚀 feat: vacation module improvements`, `🔧 fix: auth token refresh`,
`💉 hotfix: remove exposed db ports`, `🎈 release: v0.2.1-beta`.

Rules:

- **Every PR gets the emoji *and* the matching GitHub label.** State the label in your reply so it
  gets applied — the emoji alone doesn't make the PR filterable.
- Pick the prefix for what the **branch as a whole** does, not the last commit. A branch that fixes a
  bug and adds an index along the way is `fix`, even if its final commit says `feat`.
- Note the plurals: `tests` and `styles`, not `test`/`style`. `config` and `hotfix` exist here and
  have no Conventional Commits equivalent; there is no `ci` prefix (that's `chore`).
- If the repo's own workflow doc defines a different table, **that table wins** — apply it and ignore
  this one.
- Section headings carry emoji too (🔧 What / ❓ Why / 🛠️ How / 🔗 Related / ✅ Steps). Keep the body
  emoji fixed per section; only the title emoji varies by change type.
- Release PRs are `🎈 release: vMAJOR.MINOR.PATCH`, matching the release merge commit and tag.

## 5. Verification steps must be runnable

Derive commands from the repo, not from habit: read `package.json` scripts, `Makefile`, `justfile`,
`docker-compose*.yml`, `CLAUDE.md`/`AGENTS.md`. Use the repo's package manager as written in its
lockfile.

Every step names an **observable outcome**, not just an action. When the change has no UI, make the
database, queue, log line, or HTTP status the assertion target:

- `psql` / `EXPLAIN ANALYZE` for query and index work
- `redis-cli KEYS '<pattern>' | wc -l` for queue growth
- a specific log `event` field for a silent branch
- a row's column value after triggering the flow
- `<pm> test <path>` for the specific suite the change touches

Include the setup step (migration, seed, env var, local service) as step 1 when the change needs it.

## 6. Add deployment notes when the diff earns it

Most templates omit this and reviewers need it. Add a `## ⚠️ Deployment notes` section when the
branch contains:

- a schema migration — state whether it locks, backfills, or is purely additive
- a new required env var — say that startup or build fails without it
- infrastructure state the deploy does not repair on its own: an already-full queue, a stale cache,
  rows written in the old shape, jobs enqueued in the old payload format
- a change in payload/message format that old in-flight consumers or producers will encounter
- follow-up work you knowingly left out, so it isn't read as an oversight

State hazards you spotted but did not fix, plainly, as unfixed. A known-broken migration flagged in
the PR is a five-minute conversation; discovered during deploy it is an incident.

## 7. Delivery

Write the body to a scratchpad `.md` file and print it in the reply. Do not run `gh pr create` unless
asked — the author picks base branch, labels, and reviewers. When you do create it, pass the body
with `--body-file` so formatting survives.

Close your reply with anything you could not determine: the ticket number, an ambiguous prefix
choice, a hazard you flagged rather than fixed.

## Common mistakes

| Mistake | Fix |
| --- | --- |
| Diffing against `main`/`master` in a staging-flow repo | Detect the base from merged PRs (step 2) |
| Trusting `origin/HEAD` as the base | It points at the default branch, not the PR target |
| Writing from `--stat` | Read the hunks |
| Why section repeats What section | Why = root cause / requirement only |
| Ignoring an existing `PULL_REQUEST_TEMPLATE.md` | The repo's template wins over this skill |
| Emoji in the title but no label named | Both are required; state the label in your reply |
| Emoji chosen by vibe (✨, 🐛, 🚑) | Use the table — the set is fixed, not decorative |
| `test:` / `style:` / `ci:` from Conventional Commits | Plurals here: `tests`, `styles`; CI is `chore` |
| Title prefix copied from the last commit | Prefix describes the whole branch |
| Generic steps ("test the feature works") | Exact commands plus the expected observable result |
| Inventing a JIRA/GitHub issue number | Leave the placeholder |
| Silently dropping a hazard you noticed | Deployment notes, stated as unfixed |
