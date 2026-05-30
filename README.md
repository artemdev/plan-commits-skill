# plan-commits

A [Claude Code](https://claude.com/claude-code) skill that splits a large batch
of uncommitted changes into clean, feature-scoped commits — and never commits for you.

You staged 14 files across three half-finished features, a debug `out.json`, and
a `.env` you forgot about. Instead of hand-crafting `git add -p` for the next ten
minutes, you say *"split this into commits"* and review a plan.

## TL;DR

In Claude Code, add the marketplace and install the plugin (run each line on its own):

```
/plugin marketplace add artemdev/plan-commits-skill
```

```
/plugin install plan-commits@artemdev-skills
```

Then, with a messy working tree, just say:

```
split this into commits
```

Claude groups your changes, shows a plan, and hands you one `git commit` command at
a time. **You run every commit yourself.**

## The one rule

**Claude never runs `git commit`.** It reads diffs, groups files, stages each
group, and hands you a copy-pasteable command. You run it.

## Usage

Call it explicitly:

```
/plan-commits
```

or conversationally:

"commit my changes", or "split this into commits"

### Example session

```
you ▸ split this into commits

claude ▸ Read 6 changed files. Here's the plan:

         ## Proposed commits (3)

         ### 1. feat(api): add pagination to the search endpoint
         - src/api/search/pagination.ts   (new)
         - src/api/search/handler.ts      (modified)

         ### 2. refactor(ui): split the dashboard hook into smaller units
         - src/components/Dashboard/useDashboard.ts   (modified)

         ### 3. chore(repo): ignore scratch files
         - .gitignore                     (modified)

         ## ⚠️ Recommend NOT committing
         - out.json   — scratch output → add to .gitignore
         - .env       — local secrets → gitignore or move out of repo

         Approve, or tell me what to move/merge/reword.

you ▸ looks good

claude ▸ Staged commit 1. Index holds only these 2 files. Run:

           git commit -m "feat(api): add pagination to the search endpoint"

         Then say "next."

you ▸ [runs it] next

claude ▸ ✓ Commit landed. Staged commit 2. Run:

           git commit -m "refactor(ui): split the dashboard hook into smaller units"
```

You stay in control: move a file between groups, merge or split commits, reorder
them, or reword any message before you approve.

## How it works

1. **Resolves the repo root** and surveys everything uncommitted — tracked,
   untracked, deleted, renamed. Untracked files get read, not guessed from their
   names.
2. **Reads the actual diffs**, then groups by logical concern: a feature with its
   tests, foundational changes first so each commit stands on its own.
3. **Flags junk** — `*.log`, scratch JSON, `.env`, build artifacts — and
   recommends gitignoring instead of committing. It won't stage these.
4. **Writes a Conventional Commit message** per group, after checking whether your
   repo enforces a style (commitlint, husky `commit-msg` hooks) so the message
   actually passes.
5. **Shows you the whole plan** before staging anything.
6. **Walks one commit at a time**: clears the index, stages only that group's
   files, verifies nothing stray is staged, then hands you the command and stops.

## Why the index gets cleared every time

`git commit` commits whatever is staged — so a stray file left in the index from a
previous step silently rides into the wrong commit. The skill runs
`git restore --staged .` before each group and verifies the index holds *only*
that group's files. It never uses `git add .` or `git add -A`; that would defeat
the split.

## When it stays out of your way

| Situation | What happens |
|---|---|
| You want one commit of everything | It just stages and hands you one command |
| Working tree is clean | Nothing to do |
| You want Claude to commit for you | Different request — this skill is stage-and-hand-off by design |

## License

MIT — see [LICENSE](LICENSE).
