# plan-commits

A [Claude Code](https://claude.com/claude-code) skill that turns a messy working tree full of uncommitted changes into a sequence of clean, feature-scoped commits.

Claude reads your diffs, groups the changes by feature, flags junk that shouldn't be committed, stages each group, and hands you a ready-to-run `git commit` command. **It never runs `git commit` itself** — you review and commit every change yourself, keeping full control over what lands.

## Install

In Claude Code, add the marketplace:

```
/plugin marketplace add artemdev/plan-commits-skill
```

Then install the skill:

```
/plugin install plan-commits@artemdev-skills
```

That's it. The skill activates automatically when you ask Claude to help commit messy changes — e.g. *"commit my changes,"* *"split this into commits,"* *"help me commit one by one."* You can also invoke it explicitly with `/plan-commits:plan-commits`.

## What it does

- **Reads the actual diffs** (not just filenames) and groups changes by logical concern — feature + its tests together, foundational changes first.
- **Flags junk** — scratch files, logs, secrets, build artifacts — and recommends gitignoring instead of committing them.
- **Writes a Conventional Commit message** per group, and detects whether your repo enforces a commit style (e.g. [commitlint](https://commitlint.js.org)) so the message actually passes.
- **Clears the index before each group** so nothing stray rides along into the wrong commit.
- **Hands you the commit command** and stops — you commit, then tell it to continue.

## License

MIT — see [LICENSE](LICENSE).
