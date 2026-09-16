# Django Migrations — Squashing & Rebasing on a Shared Branch

Generic guide. Applies to any Django project. Placeholders to swap:
`<app_name>`, `<start_migration>`, `<end_migration>`, `<env>`.

---

# Part 1 — Squashing migrations

## What squashing is

Django writes one migration file per `makemigrations` run. Over the life of
an app that's dozens-to-hundreds of small files (`0001_initial.py`,
`0002_add_field.py`, ...). They all still get replayed on every fresh DB
(`migrate` from zero), which gets slow and clutters the migrations folder.

Squashing collapses a stable range of old migration files into one new file
that produces the *same end schema*, without replaying every intermediate
step.

**Analogy for a junior dev:** squash = zip N old migration files into 1,
keeping the end result identical. Like `git rebase -i` squashing commits —
history gets shorter, final state doesn't change.

---

## When to squash

Only a range that is **stable**: deployed to every environment (dev,
staging, prod) and nobody needs to roll back past it. Squashing a range
still being actively rolled out risks an environment being caught mid-way
through the old numbering with no matching migration to apply.

---

## Steps

### 1. Squash

```bash
cd backend
python manage.py squashmigrations <app_name> <start_migration> <end_migration>
```

Django writes one new file replacing `<start_migration>`-`<end_migration>`,
marked with `replaces = [...]` (the list of old migration names it stands
in for). **Old files stay in the folder — don't delete yet.** The
`replaces` list is what lets Django recognize "this new file already
covers what those old files did" on a DB that's already applied them.

**Verify:**
```bash
python manage.py showmigrations <app_name>
```
The new squashed migration should appear, and — on a DB that already had
the old range applied — show as applied without needing to re-run.

### 2. Confirm it produces the same schema

Run on a fresh/test DB, not on an environment holding real data:

```bash
python manage.py migrate <app_name> --plan
python manage.py test apps.<app_name>
```

**Verify:** `--plan` output ends at the same schema state as before the
squash, and the app's test suite passes exactly as it did pre-squash. If
either differs, the squash is wrong — do not deploy it.

### 3. Roll out to every environment

Deploy normally (`git pull` + `migrate`) to dev, staging, then prod, in
that order. Because the old files are still present, this is safe even on
environments that haven't picked up the squashed file yet — they just
keep applying the old ones until they do.

**Verify (per environment):**
```bash
python manage.py showmigrations <app_name> --database=<env>
```
Confirm the environment is past `<end_migration>` (either via the old
files or the new squashed one) before moving to the next environment.

### 4. Cleanup — only once every environment has applied past `<end_migration>`

```bash
# delete apps/<app_name>/migrations/<start_migration>_*.py .. <end_migration>_*.py
# then remove the `replaces = [...]` line from the new squashed migration
```

**Verify:**
```bash
python manage.py showmigrations <app_name>
```
Still shows a clean, fully-applied history with no missing-dependency
errors.

Skipping this step too early — deleting old files before every environment
has actually applied past them — breaks `migrate` on whichever environment
was still relying on the old files, since Django can no longer find the
migration it recorded as applied.

---

# Part 2 — Rebasing onto a shared branch (avoiding migration number conflicts)

## Why this happens

Each migration file's `dependencies = [(...)]` line points at the migration
that came before it in that app — that's how Django knows the order.
Numbers aren't reserved by starting work on a branch, only by what's
actually merged into the shared branch (e.g. `dev`). If two people branch
off `dev` around the same time and each run `makemigrations`, both may
independently produce a file numbered `0054_*`, since each only sees the
migrations present on `dev` at branch time. Whoever merges second ends up
colliding with the file the first person already merged.

**Analogy for a junior dev:** rebase pulls the latest shared branch under
your changes before you merge. Do it right before opening/merging your
PR — if someone else added migration `0054` while you were working, you'll
see it and can renumber yours to `0055` instead of both being `0054`.

## Steps

```bash
git fetch origin
git checkout <your-branch>
git rebase origin/<shared-branch>
```

If the rebase conflicts on a migration file itself (rare — usually it's two
*new* files, not edits to the same one, so no textual conflict), resolve
normally and continue:

```bash
git rebase --continue
```

**Verify — after rebasing, confirm your migration is still the latest leaf:**
```bash
python manage.py showmigrations <app_name>
```
Your migration should be the last one listed, with nothing else sitting at
the same number.

## If there's a number collision

If someone else's migration landed with the same number as yours (e.g.
they merged `0054_x` and your branch has `0054_y`):

1. Rename your file from `0054_y` to `0055_y`.
2. Open it and fix the `dependencies` line — it should point at their
   `0054_x` (the new latest migration on `<shared-branch>`), not whatever
   it pointed at before your rebase.

**Verify:**
```bash
python manage.py makemigrations --check --dry-run
python manage.py showmigrations <app_name>
```
`--check --dry-run` should report no missing migrations, and
`showmigrations` should show one linear chain ending in your renumbered
file — not two branches sitting at the same number.

Do this rebase **right before opening/merging the PR**, not just once at
the start of the branch — `<shared-branch>` keeps moving, so a rebase done
early can still collide with something merged afterward.

## Handling collisions (when it happens anyway)

Two migrations with the same number get merged into `<shared-branch>` —
this isn't a git problem (both branches merged cleanly, no file conflict),
it's a *migration graph* problem: two leaf migrations now claim the same
parent, and nobody rebased in time to catch it.

```bash
git pull origin <shared-branch>
python manage.py makemigrations --check --dry-run   # fails/warns if the graph has a fork
python manage.py makemigrations --merge
```

`--merge` detects the two leaf migrations, asks you to confirm, and writes
a small merge file whose only job is to join them back into one chain:

```python
# 0056_merge_20260916_1200.py
class Migration(migrations.Migration):

    dependencies = [
        ('<app_name>', '0054_add_product_type'),
        ('<app_name>', '0054_add_negative_triggers'),
    ]

    operations = []
```

**Analogy for a junior dev:** this is the migration-graph equivalent of a
`git merge` commit — it changes nothing by itself (`operations = []`), it
just has two parents instead of one, so the graph is a single line again
instead of two forks.

Rename the file to something readable
(`0056_merge_product_type_and_negative_triggers.py`) instead of leaving
the timestamp default, then commit it.

**Verify:**
```bash
python manage.py makemigrations --check --dry-run
python manage.py showmigrations <app_name>
python manage.py migrate <app_name> --plan
```
`--check --dry-run` should report nothing missing, `showmigrations` should
show one linear chain again (both `0054_*` files, then the `0056_merge_*`
file, nothing forking after it), and `--plan` should apply cleanly on a
fresh DB.

---

## Common pitfalls / Troubleshooting

| Symptom | Cause |
|---|---|
| `migrate` fails with "migration matching query does not exist" on some environment after cleanup | Old files were deleted before that environment had applied past `<end_migration>` — it was still relying on `replaces` resolution |
| `--plan` after squash shows a different operation order than before | Squash doesn't always preserve every intermediate operation faithfully (e.g. with data migrations mixed in) — treat the squash as unverified until `--plan` and tests confirm it |
| Squash includes a data migration (`RunPython`) and it silently gets skipped | `squashmigrations` can optimize away operations it thinks are redundant; data migrations with side effects need `--no-optimize` or manual review before trusting the squash |
| New squashed migration conflicts with a migration another branch added in the same numeric range | Squashing was done on a branch that diverged before teammates finished merging their own new migrations in that app — coordinate squash timing with the team, don't squash mid-flight |
| Fresh DB migrate is still slow after squashing | Squashed a range, but earlier/later ranges in the same app are still unsquashed — squashing is incremental, repeat for other stable ranges |
| Two migrations with the same number both merged into `<shared-branch>` (nobody rebased before merging) | `makemigrations` was run independently on two branches without pulling the latest shared-branch migrations first — Django doesn't reserve numbers across branches |
| `showmigrations` shows two separate leaf branches instead of one chain after a rename | Filename was renumbered but the `dependencies` line inside the file still points at the old parent — both need fixing together |
| CI passes on the branch but `migrate` fails after merge to `<shared-branch>` | The branch's own test DB never saw the teammate's migration until after merge — the numbering collision only surfaces once both are combined, so rebase immediately before merging, not just at branch-start |
| `makemigrations --check --dry-run` warns/fails on `<shared-branch>` itself (not on a feature branch) | Two leaf migrations already got merged — nobody rebased in time; resolve with `makemigrations --merge`, don't try to hand-edit `dependencies` on a file someone else already merged |
| Merge migration left named `0056_merge_20260916_1200.py` in the PR | Default timestamp name wasn't renamed before committing — rename to describe what's being joined (e.g. `0056_merge_product_type_and_negative_triggers.py`) so history stays readable |
| Merge migration accidentally has non-empty `operations` | Something was added to the file by hand instead of leaving it as the no-op join `makemigrations --merge` generated — a merge migration should only ever carry `dependencies`, any real schema change belongs in its own migration |
