---
name: commit
description: Write a clear conventional commit for the staged changes. Use when the user asks to commit, or says "write a commit message".
---

1. Run `git diff --staged`. If nothing is staged, show `git status` and ask what to stage.
2. Write a message in Conventional Commits form: `type(scope): summary` (summary under 72 chars, imperative mood).
3. Add a body only when the *why* isn't obvious from the diff.
4. Run `git commit -m "<message>"`. Never push, never use `--no-verify`.
