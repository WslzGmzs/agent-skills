---
name: pr-creator
description: 
  Create or update a GitHub PR: house-style title and body, repository template compliance, issue linking, draft state. Use when asked to "create a PR", "open a PR", "rewrite the PR description", "fix the PR title", or "polish this PR". Not for changing the code in the diff or watching CI afterwards.
compatibility: Requires Git and an authenticated GitHub CLI (gh).
---

# pr-creator

Write PR descriptions like a developer posting in Slack, not an AI summarizing a diff.

## Reference Files

| File                      | Read when                                                    |
| ------------------------- | ------------------------------------------------------------ |
| `references/pr-polish.md` | Commits contain fixup/WIP noise, the diff exceeds 500 lines, or the user asks to squash, restructure, or split |

## Workflow

Copy this checklist to track progress:

- [ ] Diff against the actual PR base (three-dot from `git merge-base HEAD origin/<base>`); never assume `main`
- [ ] Existing PR? `gh pr view --json url,state,isDraft` — OPEN means edit, not create
- [ ] Ticket ID from branch, commits, or prompt; no ID means no prefix; never invent one
- [ ] Noisy commits or >500-line diff → `references/pr-polish.md`, before pushing
- [ ] PR template present? Write the body in its shape (see Templates)
- [ ] `git push -u origin HEAD` → `gh pr create` / `gh pr edit` → return the URL

Do not ask the user to approve the description first; they can ask for a rewrite and you run `gh pr edit`.

## Rules

1. **Title.** Under 60 chars, no trailing period. With a ticket ID: `ABC-123: Add auth flow`. If the repo lints PR titles (semantic-pull-request or commitlint workflow under `.github/workflows/`), match its shape: `feat: add auth flow (ABC-123)`.
2. **Body.** One short paragraph: what changed and why it matters. Length follows the change.
3. **No fake why.** If the reason is not in the prompt, the issue, the branch, the commits, or the diff, leave it out rather than inventing one.
4. **Risk only when real.** One `Risk:` line for migrations, billing, auth, irreversible writes, wide blast radius. Otherwise nothing.
5. **Testing only if real.** Say what you ran only if you ran it. Never a `Test plan` section, never checkboxes.
6. **Partial work.** If the PR does not finish the ticket, use a non-closing reference in the body (`Part of ABC-123`, `Refs #123`), keep the ID out of the title. Closing keywords (`Fixes #123`) and Linear-ID linkage both auto-close/move the ticket on merge.
7. **Draft and reviewers only when asked.** `--draft` for "draft"/"WIP". Reviewers only for people the user named; CODEOWNERS already requests owners.
8. **No generator footers.** Never hand-write "Generated with ..." attribution, and never strip one your harness or a repo policy appends.

## Anti-patterns: never write these

- Openers that narrate the artifact: "This PR implements...", "This change ensures..."
- Changelog verbs with no reason attached: "Refactored X to improve Y", "Added comprehensive test coverage"
- Lines starting with a filename or path; the diff already lists the files
- A `Test plan` section with checkboxes
- A bullet list that restates the diff

## Example

```text
Title: PAY-482: Dedupe Stripe webhook retries

Stripe retries webhooks on timeout and our handler wasn't idempotent, so retried
events created duplicate invoices. Now we record processed event IDs and skip
repeats. Tested by replaying a captured retry sequence locally.

Risk: touches the billing write path.
```

`Risk:` earns its line (billing writes); the testing sentence exists only because a replay actually ran. A one-line fix gets a one-line body.

## Templates

GitHub reads the first `pull_request_template.md` (case-insensitive) in `.github/`, the repo root, or `docs/`, or a `PULL_REQUEST_TEMPLATE/` directory. `gh pr create --body` skips the template, so when one exists: keep its headings, answer each in a sentence or two, put the paragraph under the first heading. Add nothing the template does not ask for.

## Commands

Write the body to a temp file, pass its path via `--body-file`.

```bash
# No PR yet: create
git push -u origin HEAD
gh pr create --title "PAY-482: Dedupe Stripe webhook retries" --body-file /tmp/pr-body.md
gh pr view --json url,title   # evidence; return the url

# PR already open: edit (metadata-only — does NOT push local commits)
gh pr edit --title "..." --body-file /tmp/pr-body.md
gh pr view --json url,title
```

`gh pr edit` overwrites title and body wholesale: draft the full replacement, not a patch.

## Gotchas

- No upstream: non-interactive `gh pr create` aborts with `you must first push the current branch to a remote`. Always `git push -u origin HEAD` first.
- Non-interactive create needs both `--title` and `--body` (or `--fill`). `--fill` copies commit messages verbatim — exactly the changelog body this skill exists to avoid.
- Quote the heredoc delimiter (`<<'EOF'`). Unquoted `<<EOF` expands backticks and `$vars` in the body.
- Two failures look alike: on the default branch it's `must be on a branch named differently than "main"` (needs a branch); nothing new it's `No commits between main and <branch>` (needs a commit).
- Branch with an open PR fails with `a pull request for branch ... already exists`: `gh pr view --json state` first; OPEN → edit, MERGED/CLOSED → create new.
- Derive ticket IDs from the branch pattern (`user/abc-123-title-slug` → `ABC-123`). Never guess — tracker linkage follows the ID in the title, and a wrong one moves someone else's issue.
- Plain `git diff` omits committed changes. Use the real base with a three-dot diff, especially for stacked PRs.
- Restructure commits before the first push. Force-pushing a rewritten branch under an open PR orphans the reviewers' inline comments.
