---
name: plan-commits
description: Orchestrate splitting messy uncommitted changes into clean, feature-scoped commits. Use whenever the user wants to break up uncommitted work, stage files by feature, turn a pile of changes into atomic commits, or asks for help committing — phrases like "commit my changes", "split this into commits", "stage these by feature", "help me commit", "these files are messy", "organize my commits", or "let's commit one by one". Claude groups all uncommitted files into a plan the user reviews, then stages each group and recommends a message — but NEVER runs git commit itself. The user commits each group.
---

# /plan-commits

Turn a working tree full of uncommitted (often generated, often messy) changes into a sequence of clean, feature-scoped commits. You do the analysis and the staging; the user does every `git commit`.

## The one hard rule

**You never run `git commit` (or `git commit --amend`, or anything that writes a commit).** You stage files and hand the user a ready-to-run command. They commit themselves — that is the whole point of this workflow: the user keeps final control and reviews each commit before it lands. Staging is reversible (`git restore --staged`); a commit is not as cheap to undo. Respect that boundary even if it feels faster to just commit.

## Procedure

### 1. Work from the repo root

This skill is often invoked from a subdirectory, but changes can span the whole repo. Resolve the root first and run every git command there:

```bash
ROOT=$(git rev-parse --show-toplevel)
git -C "$ROOT" status --porcelain=v1
```

If the index already has staged changes, say so — they'll be folded into the plan. Don't silently reset the user's staging; mention it and ask if unsure.

### 2. Survey everything that changed

Look at the actual content, not just filenames — grouping by guessing from paths produces bad commits.

- `git -C "$ROOT" diff` for tracked modifications, `git -C "$ROOT" diff --stat` for the shape.
- For untracked files (`??`), read them. A new file's purpose is invisible from its name alone.
- Note deletions and renames (a rename shows as a delete + an add — keep both in the same group).

### 3. Triage for junk before grouping

Messy working trees collect files that should not be committed. Flag — don't stage — anything that looks like:

- scratch / debug output (`text.json`, `out.json`, `*.log`, `tmp/`)
- personal notes or TODO dumps (`todo.txt`, `todo/`, `notes.md`)
- secrets or local config (`.env`, `*.key`, credentials)
- build artifacts, caches, `node_modules`, coverage, generated bundles

For each, recommend the fix (usually: add to `.gitignore`, or move out of the repo) instead of putting it in a commit. If you're unsure whether something is junk or a real deliverable, ask — don't guess.

### 4. Group the rest into feature commits (file-level)

Each file goes whole into exactly one commit. Group by logical concern, using the diffs you read:

- same feature / module / route → together
- implementation + its tests → together
- a doc/spec update that describes a code change → usually with that change (especially in repos that keep docs in sync with code)
- foundational changes other commits depend on → earlier in the order, so each commit is coherent on its own

Changes to clearly separate concerns (e.g. frontend vs backend, or two distinct services/packages) go in **separate** commits unless they're trivially coupled.

### 5. Write a Conventional Commit message per group

Format: `type(scope): summary` — lowercase summary, imperative mood, **no trailing period**, header ≤ 100 chars.

- **type**: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `style`, `build`, `ci`
- **scope**: infer from the repo's own conventions — usually the package, module, or top-level area the change touches (`api`, `ui`, `auth`, `db`, ...). For repo-wide or root files use a meaningful scope (`repo`, `docs`, `ci`) or omit it. If the repo's history already follows a scope convention, match it rather than inventing one.

**Check whether the repo enforces commit messages and conform to it** — don't hardcode a style. Many repos run [commitlint](https://commitlint.js.org) via a husky `commit-msg` hook; its default `config-conventional` rejects a trailing full stop (`subject-full-stop`), upper/sentence-case subjects (`subject-case`), unknown types, and headers over 100 chars. A message the user can't actually commit wastes a round-trip. Look for `commitlint.config.*`, `.commitlintrc*`, a `commitlint` key in `package.json`, or `.husky/commit-msg`, and follow those rules. When in doubt, the safe default is plain Conventional Commits: lowercase subject, no trailing period.

Examples: `feat(api): add pagination to the search endpoint` · `refactor(ui): split the dashboard hook into smaller units` · `chore(repo): ignore scratch files`

If a group has several distinct points, add a short body with `-m` lines rather than cramming it into the summary.

### 6. Present the full plan and get approval

Show the whole plan at once so the user can see the shape before anything is staged:

```
## Proposed commits (N)

### 1. feat(api): <summary>
- src/api/search/pagination.ts        (new)
- src/api/search/handler.ts           (modified)
why: <one line>

### 2. refactor(ui): <summary>
- src/components/Dashboard/useDashboard.ts   (modified)
...

## ⚠️ Recommend NOT committing
- out.json          — looks like scratch output → add to .gitignore
- todo.txt, notes/  — personal notes → gitignore or move out of repo
```

Then ask the user to approve or adjust: move a file between commits, merge/split groups, reorder, reword a message, or change what's flagged. Apply their edits and re-show only if it changed materially.

### 7. Walk through the approved plan, one commit at a time

For each group, in order:

1. **Clear the index first.** `git commit` commits whatever happens to be staged — so if a previous group or a stray edit left something in the index, it silently rides along into this commit. Start each group from an empty index: `git -C "$ROOT" restore --staged .` (unstages everything; never changes file content). Skipping this is the single most common way this skill produces a wrong commit: a stray already-staged file silently rides along into the first group's commit because the index wasn't cleared first.
2. Stage exactly that group's files: `git -C "$ROOT" add -- <file1> <file2> ...`. `git add` stages new, modified, and deleted paths.
3. Confirm the index holds **only** this group: `git -C "$ROOT" status --short`. Read it actively — every line whose *first* column is `A`/`M`/`D`/`R` is staged and will be committed. Each such line must be a file from this group; if anything extra is staged, you missed step 1, so unstage it before continuing. Don't just check that your files are present — check that nothing else is.
4. Hand over the command, copy-pasteable:

   ```
   git commit -m "feat(api): add pagination to the search endpoint"
   ```

5. **Stop. Hand control back to the user** with a clear "run that, then tell me to continue (or edit the message however you like)." Do not stage the next group, and do not commit.

### 8. Resume after the user commits

When the user comes back:

- Verify the commit landed: `git -C "$ROOT" log -1 --stat` and `git -C "$ROOT" status --short`. The group's files should now be gone from the working set.
- If they're still staged/unstaged, the commit didn't happen (or the message was rejected) — say so and re-offer the command rather than barging ahead.
- Then stage the next group and repeat from step 7. Follow the approved plan, but if the working tree changed since (the user edited a file), adjust that group and mention it.

When the plan is done, give a one-line summary: how many commits landed, and any flagged files still sitting in the working tree for the user to deal with.

## Guardrails

- **Never commit.** Worth repeating. You stage; the user commits.
- **The index must contain only the current group.** Clear it (`git restore --staged .`) before staging each group, then add explicit paths. Never `git add .` or `git add -A` across the whole tree — that defeats the split. After staging, verify nothing extra is staged, since the commit takes the whole index regardless of what you intended.
- **Don't delete or move the user's files** to "clean up." Flag junk and recommend; let the user act.
- **Don't reset someone's existing staged work** without flagging it first.
- **Read diffs before grouping.** A plan built from filenames alone is a guess.

## When to skip this skill

- The user explicitly wants a single commit of everything — just stage and hand over one command.
- There's nothing uncommitted (`git status` is clean).
- The user wants you to commit for them — that's a different request; this skill is specifically for the stage-and-hand-off flow.
