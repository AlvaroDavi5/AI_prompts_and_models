---
name: commit-message
description: Generate a Conventional Commits / Semantic Commits message from the current changes. Looks at staged files first, falls back to the working tree, then to a diff against main/master if nothing is staged or modified. Picks a valid type and optional scope (TASK-ID, deps, dev...), writes a header + body, and signs off with the user and Claude as co-author. Use when the user asks to write, suggest, or generate a commit message, or asks to commit.
argument-hint: (optional) ticket ID or scope hint
user-invocable: true
---

# Commit Message

This skill follows this repo's `commit-message.rule.md` (loaded as project/global context) for
format, allowed types, scope extraction, and length limits. This skill adds what that rule doesn't
cover: **where to source the diff from** when nothing is staged, and the **sign-off/co-author
footer**.

## When to Use
- The user asks to write/suggest a commit message.
- The user asks to commit changes (draft the message first, get approval, then stage/commit per
  the git safety rules — never commit without explicit confirmation).

## Procedure

### Step 1 — Pick the change source (in priority order)
1. **Staged changes**: `git diff --staged --name-only`. If non-empty, use `git diff --staged` as
   the diff to analyze. This is the preferred source — it's what will actually be committed.
2. **Working tree**: if nothing is staged, check `git status --porcelain` for modified/untracked
   tracked files. If non-empty, use `git diff` (unstaged) as the diff to analyze. Tell the user
   these changes aren't staged yet — the message is a preview, not commit-ready.
3. **Diff against main/master**: if the working tree is clean too, find the default branch
   (`git symbolic-ref refs/remotes/origin/HEAD` or fall back to checking for `main` then `master`),
   then diff the current branch against it: `git diff <default-branch>...HEAD`. Use this to
   summarize what the whole branch would introduce if merged.
4. If all three are empty, stop and tell the user there are no changes to describe.

Always state which source was used, since it changes what the message actually represents.

### Step 2 — Analyze the diff
From the chosen diff, identify (per the commit-message rule):
- What changed: new/deleted/renamed files, modified logic, added/removed dependencies.
- Why it changed, when inferable from context (bug fix, feature, refactor...).
- Affected modules/layers (controller, service, repository, config, tests, etc.).

### Step 3 — Determine type and scope
- Type: one of `feat`, `fix`, `perf`, `refactor`, `test`, `docs`, `style`, `build`, `ci`, `chore`,
  `revert` — per the allowed-types table in the commit-message rule. Pick the one matching the
  dominant nature of the change; if changes are mixed, pick the type of the most significant part
  and mention the rest in the body.
- Scope: run `git rev-parse --abbrev-ref HEAD`. If the branch name contains `[A-Z]+-[0-9]+`
  (e.g. `DEV-123`), use it as the task ID/scope. If the branch has no such pattern, ask the user
  whether there's a task ID to use as scope before drafting the header — don't silently omit it or
  guess one. If the user confirms there isn't one, scope is optional — use a short descriptive
  area (`deps`, `auth`, `no-release`...) only if it adds clarity, or omit it.

### Step 4 — Draft the message
- **Header**: `<type>(<scope>): <short description>` — lowercase, imperative mood, no trailing
  period, ≤ 100 chars.
- **Body** (for non-trivial changes): explain what and why per file/module, based on the actual
  diff content — not just filenames. Lines ≤ 200 chars.
- **Footer**:
  - Issue refs / `BREAKING CHANGE:` when applicable, lines ≤ 150 chars.
  - Always ask the user whether to include `Signed-off-by`/`Co-Authored-By` lines before adding
    them — never add them silently. If they agree:
    - `Signed-off-by: <name> <email>` — get `<name>`/`<email>` from `git config user.name` and
      `git config user.email`.
    - `Co-Authored-By: Claude <noreply@anthropic.com>`

### Step 5 — Present and ask to commit
Show the drafted message to the user, then ask whether to commit it as-is.
- If the user agrees: stage the relevant files if they aren't already staged (confirm which files
  first if the source was the working tree or a branch diff), then run `git commit` with the
  **entire** message exactly as displayed (header, body, and footer) — use a heredoc so formatting
  and line breaks are preserved, don't retype or paraphrase it.
- If the user declines, or the source required staging that wasn't confirmed: only display the
  message. Do not run `git add` or `git commit`.

## Decision Points
- No staged, unstaged, or branch-ahead changes anywhere → stop, tell the user there's nothing to
  describe.
- Mixed change types in one diff → pick the dominant type, call out the rest in the body, and
  suggest splitting into multiple commits if the mix is large/unrelated.
- On `main`/`master` with no upstream ticket pattern → ask the user for a task ID rather than
  inventing or omitting one outright.

## Completion Criteria
- [ ] Change source identified and stated (staged / working tree / branch diff).
- [ ] Diff actually analyzed — type, scope, and body reflect real content, not just file names.
- [ ] Task ID/scope taken from the branch name, or explicitly asked about if the branch has none.
- [ ] User asked whether to include `Signed-off-by`/`Co-Authored-By` before adding either.
- [ ] Header follows format rules (lowercase, imperative, no period, ≤ 100 chars).
- [ ] Body (if present) explains what/why, lines ≤ 200 chars.
- [ ] Footer includes issue refs/breaking changes if applicable (≤ 150 chars/line), plus
      `Signed-off-by`/`Co-Authored-By: Claude` only if the user opted in.
- [ ] User explicitly asked whether to commit the displayed message.
- [ ] If agreed: committed with the exact displayed message (header+body+footer), no retyping.
- [ ] If declined: message only displayed — no `git add`/`git commit` run.
