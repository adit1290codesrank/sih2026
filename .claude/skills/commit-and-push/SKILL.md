---
name: commit-and-push
description: Commit and push the current work for the team. Use when the user says "commit", "push", "commit and push", or /commit-and-push. Works on any branch; always confirms the target branch, and shows the message before committing.
---

# Commit and push

1. **Confirm the branch.** Run `git branch --show-current` and `git rev-parse --abbrev-ref @{u}` (it may have no upstream). Ask the user which branch to push to, and never assume it. Switch branches only if they say so.

2. **Check scope.** Run `git status --short` and compare each path with the Team workflow section of `CLAUDE.md`.
   - Web branch (e.g. `romir-web`): only `web/**` is in scope.
   - `site/` is static data. Stage it only with the user's explicit OK.
   - Main-owned files (`src/ scripts/ docs/ results/ figures/ configs/ tests/ CLAUDE.md STEPS.md README.md`) must not change on a web branch. List them and leave them unstaged.
   - Never stage `shards/ work/ runs/ ext/ *.pt`.

3. **Stage by explicit path**, never `git add -A` or `git add .`. If `src/` or `tests/` changed, run `pytest -q` first and stop if it fails.

4. **Write the message like a teammate would:**
   - one short subject line, under 60 characters, plain words, no trailing period;
   - a 1–2 line body only if the subject can't carry it.

   Never include:
   - `Co-Authored-By` or any other trailer;
   - any mention of Claude, AI, or a model;
   - emoji or bullet lists.

   Good: `add website brief for the design pass`. Bad: `feat: Comprehensive website brief document outlining...`

5. **Show the user the final message, the staged files and the target branch, then wait for approval.** Edit the message if asked.

6. Commit with `git commit -m "<subject>"` (add `-m "<body>"` if there is one), then push with `git push -u origin <confirmed branch>`. Never force-push.

7. If the push is rejected, run `git pull --rebase origin <branch>` and push again. On conflicts, stop and ask.
