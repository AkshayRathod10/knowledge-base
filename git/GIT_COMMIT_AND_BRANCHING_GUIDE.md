# Git Commit Messages, Branch Naming, and Workflow — Industry Standard

Generic guide. Applies to any project regardless of language or hosting
(GitHub, GitLab, Bitbucket). Placeholders to swap: `<ticket-id>`,
`<short-description>`, `<version>`.

---

## Commit messages — Conventional Commits

Use the [Conventional Commits](https://www.conventionalcommits.org) format.
It's the de facto industry standard: readable by humans, parseable by
tooling (changelog generators, semantic-release, commit linters).

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Example:**
```
fix(auth): reject expired refresh tokens on renewal

Previously an expired refresh token would silently issue a new access
token. Now the renewal endpoint returns 401 and the client is forced
to re-authenticate.

Closes #482
```

### Types

| Type | Use for |
|---|---|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `style` | Formatting, whitespace, missing semicolons — no code behavior change |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or correcting tests |
| `build` | Build system or external dependencies (webpack, npm, docker) |
| `ci` | CI configuration/scripts |
| `chore` | Everything else (tooling, repo maintenance) — not shipped to users |
| `revert` | Reverts a previous commit |

### Rules

- **Subject line**: imperative mood ("add", not "added"/"adds"), lowercase
  after the colon, no trailing period, ideally ≤50 characters, hard cap 72.
  Test: the subject should complete "If applied, this commit will
  **`<subject>`**."
- **Scope** (optional): the component/module affected — `feat(api): ...`,
  `fix(cli): ...`. Omit if it doesn't add clarity.
- **Body** (optional but expected for non-trivial changes): explains *why*,
  not *what* — the diff already shows what changed. Wrap at ~72 chars.
  Blank line between subject and body.
- **Footer**: `BREAKING CHANGE: <description>` for breaking changes (also
  add `!` after the type/scope, e.g. `feat(api)!: ...`), and issue
  references (`Closes #123`, `Refs #123`).
- One logical change per commit. If the commit message needs "and" to
  describe it, it's probably two commits.

### Enforcing it — commit-msg hook

```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg (or via husky/commitlint in a Node project)
pattern='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9-]+\))?!?: .{1,72}$'
if ! head -1 "$1" | grep -qE "$pattern"; then
  echo "Commit message doesn't follow Conventional Commits format." >&2
  echo "Expected: <type>(<scope>): <subject>" >&2
  exit 1
fi
```

**Verify:**
```bash
echo "bad message" | git commit --file=- --dry-run
```
Should be rejected; `git commit -m "fix(api): handle null response"` should pass.

For JS/TS projects, the standard tooling is
[`@commitlint/cli`](https://commitlint.js.org) + `@commitlint/config-conventional`,
wired in via [Husky](https://typicorn.github.io/husky/)'s `commit-msg` hook.

---

## Branch naming

```
<type>/<ticket-id>-<short-description>
```

| Type | Use for | Example |
|---|---|---|
| `feature/` | New functionality | `feature/JIRA-123-oauth-login` |
| `fix/` | Bug fix | `fix/JIRA-456-null-pointer-on-logout` |
| `hotfix/` | Urgent production fix, branched from a release tag/main | `hotfix/JIRA-789-payment-timeout` |
| `release/` | Release stabilization branch | `release/2.4.0` |
| `chore/` | Tooling, deps, repo maintenance | `chore/upgrade-node-20` |
| `docs/` | Documentation only | `docs/update-deploy-runbook` |

Rules:
- All lowercase, words separated by hyphens (kebab-case) — not
  underscores or camelCase. Git refs are case-sensitive; mixed case
  invites typos and duplicate-looking branches.
- Keep the description short (3–5 words) — it's a label, not a commit
  message. The ticket ID is the source of truth for full context.
- No `<ticket-id>`? Use a short slug instead: `fix/logout-null-pointer`.
  Don't invent a fake ticket number.
- Never branch names with spaces, or characters `~ ^ : ? * [ \`.

---

## Branching model: trunk-based vs. Git Flow

| | Trunk-based (short-lived branches) | Git Flow (`develop` + release branches) |
|---|---|---|
| Branch lifetime | Hours to a few days | Weeks, until release |
| Merge target | Directly to `main` | `develop`, then `main` on release |
| Best for | Continuous deployment, web services, small-to-mid teams | Scheduled/versioned releases, multiple versions in production simultaneously (e.g. installed software, libraries with LTS versions) |
| Overhead | Low — fewer long-lived branches to keep in sync | Higher — `develop`/`main` drift, more merge conflicts the longer a branch lives |
| CI/CD fit | Matches CI/CD naturally — every merge to `main` is releasable | Needs an explicit release branch step before `main` gets updated |

**Default recommendation: trunk-based development with short-lived
feature branches**, merged to `main` behind feature flags if the work
needs to land before it's ready to ship. This is the direction the
industry has moved (Google, GitHub itself, most SaaS companies) because
long-lived branches accumulate merge conflicts and drift from `main`
faster than teams keep them in sync. Reach for Git Flow specifically
when you need to support multiple released versions concurrently —
not by default.

Either model: **branch from an up-to-date `main`**, and merge back
promptly. A feature branch older than ~1 week is a sign to either merge
part of it (behind a flag) or rebase it onto current `main`.

---

## Merging: squash, merge commit, or rebase

| Strategy | History result | When |
|---|---|---|
| **Squash merge** | One commit per PR on `main`, linear history | Default for most feature/fix branches — keeps `main` readable, one entry per unit of work |
| **Merge commit** (`--no-ff`) | Preserves every commit + a merge commit marking the branch | When the individual commits on the branch are independently meaningful and worth keeping (e.g. a large feature built as reviewed, working increments) |
| **Rebase and merge** | Branch commits replayed onto `main`, no merge commit | When you want linear history *and* to preserve individual commits — requires commits on the branch are already clean (see squash-locally note below) |

**Never rewrite history that's already been pushed and could be in use
by others** (no `git push --force` to `main`/shared branches, no
`rebase` of commits others have already pulled) — this is the one rule
that's non-negotiable regardless of which strategy a team picks.
Force-pushing your own not-yet-reviewed feature branch to clean up
local history before opening a PR is fine; force-pushing after review
has started requires warning your reviewer (their in-progress comments
can become orphaned).

If squashing locally before merge (as opposed to a platform's "Squash
and merge" button), the squashed commit message should read like a
well-formed Conventional Commit summarizing the PR, not a concatenation
of the WIP commit log:
```bash
git rebase -i main   # mark WIP commits `squash`/`fixup`, rewrite the message
```

---

## Tags and versioning

Use [Semantic Versioning](https://semver.org): `vMAJOR.MINOR.PATCH`.

- **MAJOR** — breaking change
- **MINOR** — backward-compatible new functionality
- **PATCH** — backward-compatible bug fix

```bash
git tag -a v2.4.0 -m "v2.4.0"
git push origin v2.4.0
```

Use annotated tags (`-a`), not lightweight tags — annotated tags carry
a message, tagger, and date, and are what `git describe` expects.

**Verify:**
```bash
git tag -n9 v2.4.0
```
Should show the tag message, not just the tag name.

---

## Pull requests

- PR title should itself be a valid Conventional Commit subject when the
  merge strategy is squash — it becomes the commit message on `main`.
- Keep PRs small enough to review in one sitting. A PR that touches
  20+ files across unrelated concerns should usually have been two PRs.
- Link the ticket/issue in the PR description, not just the branch name
  — branch names aren't shown in most "what changed and why" views.
- Don't merge your own PR without review unless the repo's rules
  explicitly allow it (solo repos, documented emergency-fix exception).

---

## Common pitfalls

| Symptom | Cause |
|---|---|
| Changelog tooling (semantic-release, etc.) doesn't pick up a change | Commit type doesn't match Conventional Commits (typo in `feat`/`fix`, or wrong case) |
| `main` history is a wall of "wip", "fix typo", "address review comments" | Squash merge wasn't used, or commits weren't cleaned up before merging with a merge-commit/rebase strategy |
| Two branches named almost the same but differ by case (`Feature/x` vs `feature/x`) | Branch naming wasn't lowercased — Git refs are case-sensitive, some filesystems (macOS default, Windows) are not, causing checkout confusion |
| Force-push broke a teammate's local branch | History was rewritten on a branch already pulled by someone else |
| `git tag -n` shows nothing for a tag | Tag was created lightweight (`git tag v1.0.0`) instead of annotated (`git tag -a v1.0.0 -m "..."`) |
| Merge conflict storm on a feature branch | Branch lived too long without rebasing/merging from `main` |
