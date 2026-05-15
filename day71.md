# 📅 Day 71 — Thursday, 24 July 2026
# 🔀 Git Advanced: Rebasing, Cherry-pick, Conflict Resolution

---

## 🎯 Today's Goal

You've been using Git basics (commit, push, pull, branch) for 10 weeks. Today you level up to professional Git — the skills that separate "I use Git" from "I'm comfortable collaborating on a team." These are also common interview topics for data engineering roles.

**Philosophy:** Git mistakes are scary ("I lost my code!"). But Git is designed to be safe — almost everything is recoverable. Today you learn the advanced tools AND how to undo mistakes. By the end, you'll never fear Git again.

---

## ☀️ Morning Block (2 hours): Rebase vs Merge + Interactive Rebase

### Git Review (Quick — 15 minutes)

You should already know:
- `git add`, `git commit`, `git push`, `git pull`
- `git branch`, `git checkout`, `git merge`
- `git log`, `git status`, `git diff`

If any of these feel shaky, spend 15 minutes on [learngitbranching.js.org](https://learngitbranching.js.org) before continuing.

### The Problem: Divergent Branches

```
You're on main branch. Your colleague pushed new commits.
You're on feature branch with your own commits.
Now you need to combine them.

main:     A --- B --- C --- D (colleague's work)
                    \
feature:              E --- F (your work)
```

Two ways to combine: **merge** or **rebase**.

### Merge: The Safe Way

```bash
git checkout main
git merge feature
```

Result:
```
main:     A --- B --- C --- D --- M (merge commit)
                    \             /
feature:              E --- F ---
```

**Pros:** Preserves history exactly as it happened. Safe. Reversible.
**Cons:** Creates a merge commit. History can get messy with many branches.

### Rebase: The Clean Way

```bash
git checkout feature
git rebase main
```

Result:
```
main:     A --- B --- C --- D
                          \
feature:                   E' --- F' (your commits replayed on top)
```

**Pros:** Linear history. No merge commits. Clean git log.
**Cons:** Rewrites history (changes commit hashes). Dangerous if shared.

### When to Use Each

| Scenario | Use | Why |
|----------|-----|-----|
| Updating your feature branch with latest main | **Rebase** | Keeps your work on top, clean history |
| Merging a completed feature into main | **Merge** | Preserves history, safe for shared branch |
| Syncing with remote before push | **Rebase** | Avoids unnecessary merge commits |
| Multiple people on same branch | **Merge** | Never rebase shared branches |

**Golden Rule:** Never rebase commits that have been pushed to a shared branch.

### Exercise 1: Rebase Practice

Create a test repo and practice:

```bash
mkdir git-practice && cd git-practice
git init

# Create main commits
echo "v1" > file.txt
git add . && git commit -m "A: initial commit"

echo "v2" >> file.txt
git add . && git commit -m "B: second commit"

# Create feature branch
git checkout -b feature

echo "feature work" > feature.txt
git add . && git commit -m "E: feature work"

echo "more feature" >> feature.txt
git add . && git commit -m "F: more feature work"

# Meanwhile, main got new commits
git checkout main
echo "v3" >> file.txt
git add . && git commit -m "C: main update"

echo "v4" >> file.txt
git add . && git commit -m "D: another main update"

# Now rebase feature onto main
git checkout feature
git rebase main

# Check the clean history
git log --oneline --graph
```

<details>
<summary>🔑 Expected git log output after rebase</summary>

```
* f1a2b3c (HEAD -> feature) F: more feature work
* e4d5c6b E: feature work
* d3c4b5a D: another main update
* c2b3a4d C: main update
* b1a2d3e B: second commit
* a0b1c2d A: initial commit
```

Notice: linear history, no merge commit. Your E and F commits are now on top of D.
</details>

### Interactive Rebase: Edit History

```bash
git rebase -i HEAD~3
```

This opens an editor showing your last 3 commits:
```
pick e4d5c6b E: feature work
pick f1a2b3c F: more feature work
pick a1b2c3d G: yet another commit
```

You can change `pick` to:
- **squash** — combine with previous commit
- **reword** — change commit message
- **edit** — pause to modify the commit
- **drop** — remove the commit entirely
- **reorder** — move commits up/down

### Exercise 2: Squash Commits

Squash your last 3 commits into one:

```bash
git rebase -i HEAD~3
# Change to:
# pick e4d5c6b E: feature work
# squash f1a2b3c F: more feature work
# squash a1b2c3d G: yet another commit
```

<details>
<summary>🔑 What happens</summary>

Git opens another editor asking for the combined commit message. Write something like:
```
Add feature: complete implementation

- Initial feature work
- Additional improvements
- Final fixes
```

Now `git log --oneline` shows ONE commit instead of three. This is how you clean up "WIP", "fix typo", "actually fix" commit chains before pushing.
</details>

---

## 🌤️ Afternoon Block (2 hours): Cherry-pick, Conflicts, and Undo

### Cherry-pick: Grab Specific Commits

```bash
# You want commit abc1234 from another branch, but not the whole branch
git cherry-pick abc1234
```

**Use case:** A colleague fixed a bug on their branch. You need that fix on your branch NOW, but can't wait for them to merge.

```bash
# Scenario: hotfix on main, you need it on your feature branch
git checkout feature
git cherry-pick a1b2c3d  # the hotfix commit hash

# Cherry-pick a range
git cherry-pick a1b2c3d..e4f5g6h  # exclusive of first, inclusive of last
```

### Exercise 3: Cherry-pick Practice

```bash
# Create a bugfix on main
git checkout main
echo "bugfix" > fix.txt
git add . && git commit -m "HOTFIX: critical bug fix"

# Get the commit hash
git log --oneline -1
# e.g., a1b2c3d HOTFIX: critical bug fix

# Apply it to feature branch without merging everything
git checkout feature
git cherry-pick a1b2c3d
```

<details>
<summary>🔑 Result</summary>

The fix.txt file now exists on your feature branch, without pulling in any other changes from main. The commit gets a NEW hash (because it's a new commit with the same changes).
</details>

### Merge Conflicts: Don't Panic

Conflicts happen when two branches modify the same line. Git can't decide which version to keep — so it asks you.

```
<<<<<<< HEAD (your changes)
SELECT * FROM orders WHERE status = 'delivered';
=======
SELECT * FROM orders WHERE status = 'completed';
>>>>>>> feature (their changes)
```

### Conflict Resolution Strategy

1. `git status` — see which files have conflicts
2. Open each file — look for `<<<<<<<`, `=======`, `>>>>>>>` markers
3. Decide: keep yours, keep theirs, or combine both
4. Remove the markers
5. `git add <file>` — mark as resolved
6. `git commit` (for merge) or `git rebase --continue` (for rebase)

### Exercise 4: Create and Resolve a Conflict

```bash
# On main
git checkout main
echo "SELECT * FROM orders WHERE status = 'pending';" > query.sql
git add . && git commit -m "Query: pending orders"

# On feature, edit SAME line
git checkout -b conflict-practice
echo "SELECT * FROM orders WHERE status = 'delivered';" > query.sql
git add . && git commit -m "Query: delivered orders"

# Back to main, edit SAME line
git checkout main
echo "SELECT * FROM orders WHERE status = 'cancelled';" > query.sql
git add . && git commit -m "Query: cancelled orders"

# Try to merge — CONFLICT!
git merge conflict-practice
```

<details>
<summary>🔑 Resolution</summary>

```bash
# Git will report: CONFLICT in query.sql
# Open query.sql, you'll see:
# <<<<<<< HEAD
# SELECT * FROM orders WHERE status = 'cancelled';
# =======
# SELECT * FROM orders WHERE status = 'delivered';
# >>>>>>> conflict-practice

# Decide: maybe you want BOTH queries
echo "SELECT * FROM orders WHERE status = 'cancelled';
SELECT * FROM orders WHERE status = 'delivered';" > query.sql

# Mark as resolved
git add query.sql
git commit -m "Merge: combine cancelled and delivered queries"
```

**Pro tip for data engineers:** When two people edit the same SQL file, the most common resolution is to keep BOTH queries (they're usually different analyses, not conflicting edits).
</details>

### Git Stash: Temporary Storage

```bash
# You're mid-work but need to switch branches urgently
git stash                # Save current changes temporarily
git checkout other-branch  # Do something else
git checkout my-branch     # Come back
git stash pop              # Restore your changes

# List stashes
git stash list

# Stash with message
git stash save "WIP: MakanExpress ETL script"
```

### Git Reset: Undo Commits

```bash
# Undo last commit, keep changes in working directory (safest)
git reset --soft HEAD~1

# Undo last commit, keep changes staged
git reset --mixed HEAD~1   # same as git reset HEAD~1

# Undo last commit, DELETE changes (DANGEROUS)
git reset --hard HEAD~1    # only use if you're sure
```

| Command | Commit History | Staging Area | Working Directory |
|---------|---------------|--------------|-------------------|
| `--soft` | Undone | Kept | Kept |
| `--mixed` | Undone | Undone | Kept |
| `--hard` | Undone | Undone | Deleted |

### Git Reflog: The Safety Net

```bash
# "Oh no, I accidentally reset --hard!"
git reflog
# Shows EVERYTHING you've done:
# a1b2c3d HEAD@{0}: reset: moving to HEAD~1
# e4f5g6h HEAD@{1}: commit: important work
# ...

# Recover!
git reset --hard e4f5g6h  # go back to before the reset
```

**Even `--hard` resets are recoverable with reflog (for ~30 days).** This is why Git is safe.

### Exercise 5: Undo and Recover

```bash
# Make an important commit
echo "important data" > important.txt
git add . && git commit -m "IMPORTANT: do not lose this"
git log --oneline -1  # note the hash

# "Accidentally" hard reset
git reset --hard HEAD~1

# Oh no! File is gone!
ls important.txt  # No such file

# Recover with reflog
git reflog  # find the hash
git reset --hard <hash>  # recover

# File is back!
ls important.txt  # important data
```

---

## 🌙 Evening (1 hour): Best Practices + Quick Reference

### Git Best Practices for Data Teams

1. **Commit often, push regularly** — small commits are easier to review and revert
2. **Write meaningful messages** — "fix" is useless. "Fix: handle null values in order_amount column" is useful
3. **Use branches for everything** — never commit directly to main
4. **Pull/rebase before push** — avoid unnecessary merge conflicts
5. **Review before merging** — even your own PRs

### Commit Message Convention (Conventional Commits)

```
type(scope): description

feat(etl): add MakanExpress daily load pipeline
fix(sql): handle null values in order_amount
refactor(dbt): consolidate staging models
docs(readme): add setup instructions
test(quality): add data validation for orders table
chore(deps): update psycopg2 to 2.9.9
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`
Scope: the area of code (etl, sql, dbt, airflow, etc.)

### 🗺️ Quick Reference Card

```
=== REBASING ===
git rebase main              # Rebase current branch onto main
git rebase -i HEAD~3         # Interactive rebase last 3 commits
git rebase --continue        # Continue after resolving conflict
git rebase --abort           # Cancel rebase, go back to before

=== CHERRY-PICK ===
git cherry-pick abc1234      # Apply specific commit to current branch

=== CONFLICTS ===
git status                   # See conflicted files
# Edit files, remove <<<< ==== >>>> markers
git add <file>               # Mark as resolved
git commit                   # Complete merge
git rebase --continue        # Complete rebase

=== UNDO ===
git stash                    # Temporarily save changes
git stash pop                # Restore stashed changes
git reset --soft HEAD~1      # Undo commit, keep everything
git reset --hard HEAD~1      # Undo commit, delete changes
git reflog                   # See all operations (safety net)

=== USEFUL ALIASES ===
git log --oneline --graph --all    # Visual branch history
git log --oneline -10              # Last 10 commits
git diff --staged                  # See what's about to be committed
```

### 📝 Today's Checklist

- [ ] Completed rebase exercise (clean linear history)
- [ ] Completed interactive rebase (squash commits)
- [ ] Completed cherry-pick exercise
- [ ] Created and resolved a merge conflict
- [ ] Used git stash to save and restore work
- [ ] Practiced git reset (soft, mixed, hard)
- [ ] Used reflog to recover from a "mistake"
- [ ] Updated commit messages to follow convention
- [ ] Pushed any changes to GitHub

---

*Day 71 complete! Tomorrow: GitHub Actions CI/CD — automate your pipeline testing.* 🚀
