# Student/Instructor Branch Separation -- Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Set up a two-branch model where `main` is student-facing only and `instructor` contains all content, with a GitHub Action that auto-syncs.

**Architecture:** Rename current `main` to `instructor`. A GitHub Action on push to `instructor` checks out the branch, deletes files listed in `.admin-files`, and force-pushes the result to `main`. Update CLAUDE.md to reflect the new workflow.

**Tech Stack:** GitHub Actions, bash, git

---

## Current State

- `main` branch has all content (student + admin)
- No `.github/` directory exists
- No `instructor` branch exists
- CLAUDE.md git workflow references `main` as the target for PRs
- Feature branches are created off `main`

## Admin Files to Strip

These patterns (from `.admin-files` manifest) will be removed from `main`:

```
# Video scripts (per-lesson)
**/script.md

# Course planning
lessons/clab/COURSE_PLAN.md
lessons/clab/VIDEO_SCRIPT_TEMPLATE.md

# AI assistant instructions
CLAUDE.md

# Design docs and plans
docs/plans/
```

---

### Task 1: Create the `.admin-files` manifest

**Files:**
- Create: `.admin-files`

**Step 1: Create the manifest file**

Create `.admin-files` in the repo root with one path/glob pattern per line. Lines starting with `#` are comments. This file lives on the `instructor` branch and tells the GitHub Action what to strip.

```
# .admin-files -- Patterns stripped from main (student view)
# One pattern per line. Lines starting with # are comments.
# This file itself is also stripped from main.

# This manifest
.admin-files

# Video recording scripts (in every lesson directory)
**/script.md

# Course-level planning files
lessons/clab/COURSE_PLAN.md
lessons/clab/VIDEO_SCRIPT_TEMPLATE.md

# AI assistant instructions
CLAUDE.md

# Design documents and implementation plans
docs/plans/
```

**Step 2: Verify patterns match expected files**

```bash
# Test each pattern against the working tree
while IFS= read -r pattern; do
  [[ "$pattern" =~ ^#.*$ || -z "$pattern" ]] && continue
  echo "--- $pattern ---"
  git ls-files "$pattern" 2>/dev/null || find . -path "./$pattern" -o -path "./$pattern*" 2>/dev/null | head -5
done < .admin-files
```

Verify: each pattern matches the expected admin files and nothing else.

**Step 3: Commit**

```bash
git add .admin-files
git commit -m "chore: add .admin-files manifest for branch separation"
```

---

### Task 2: Create the GitHub Action

**Files:**
- Create: `.github/workflows/sync-student-branch.yml`

**Step 1: Create the workflow directory**

```bash
mkdir -p .github/workflows
```

**Step 2: Write the workflow file**

Create `.github/workflows/sync-student-branch.yml`:

```yaml
# Syncs the instructor branch to main, stripping admin-only files.
# Triggered on every push to the instructor branch.
#
# The .admin-files manifest (in repo root) lists patterns to remove.
# main = student-facing view (clean, no instructor content)
# instructor = full working branch (student + instructor content)

name: Sync student branch

on:
  push:
    branches:
      - instructor

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout instructor branch
        uses: actions/checkout@v4
        with:
          ref: instructor
          fetch-depth: 0

      - name: Remove admin-only files
        run: |
          # Read .admin-files manifest and remove matching files
          while IFS= read -r pattern; do
            # Skip comments and blank lines
            [[ "$pattern" =~ ^[[:space:]]*#.*$ || -z "${pattern// }" ]] && continue

            # Use git ls-files for glob patterns, fall back to find
            files=$(git ls-files "$pattern" 2>/dev/null)
            if [ -z "$files" ]; then
              files=$(find . -path "./$pattern" -o -path "./$pattern*" 2>/dev/null | sed 's|^\./||')
            fi

            if [ -n "$files" ]; then
              echo "Removing pattern '$pattern':"
              echo "$files" | while IFS= read -r f; do
                echo "  - $f"
                rm -rf "$f"
              done
            fi
          done < .admin-files

          # Also remove the workflow itself and .admin-files from the student branch
          rm -rf .github/workflows/sync-student-branch.yml
          rm -f .admin-files

          # Clean up empty directories
          find . -type d -empty -not -path './.git/*' -delete 2>/dev/null || true

      - name: Force-push to main
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git add -A
          git commit -m "sync: update student branch from instructor" --allow-empty

          git push --force origin HEAD:main
```

**Step 3: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/sync-student-branch.yml')); print('Valid YAML')"
```

**Step 4: Commit**

```bash
git add .github/workflows/sync-student-branch.yml
git commit -m "ci: add GitHub Action to sync instructor -> main (strip admin files)"
```

---

### Task 3: Update CLAUDE.md for new workflow

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Update the Content Audience Convention section**

Replace the current text (around line 33):

```
The `main` branch contains both student-facing and instructor-only content. The convention:
```

With:

```
The repo uses two branches:

- **`instructor`** (working branch) -- contains all content (student + instructor)
- **`main`** (student view) -- auto-generated, contains only student-facing content

A GitHub Action (`.github/workflows/sync-student-branch.yml`) syncs `instructor` to `main` on every push, stripping files listed in `.admin-files`.
```

**Step 2: Update the Git Workflow section**

Replace the current workflow (around lines 51-66) to reference `instructor` instead of `main`:

Change:
- "Never commit directly to main" -> "Never commit directly to instructor"
- `gh pr create --base main` -> `gh pr create --base instructor`
- Feature branch example: `git checkout -b feature/your-feature-name` (from instructor)

Updated workflow block:

```bash
# Standard workflow
git checkout instructor
git checkout -b feature/your-feature-name
# ... make changes ...
git add <specific-files>
git commit -m "Descriptive message"
git push -u origin feature/your-feature-name
gh pr create --base instructor --title "Title" --body "Description"
```

**Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for instructor/main branch model"
```

---

### Task 4: Rename main to instructor and create new main

This is the migration step. It renames the current `main` to `instructor` on GitHub, then lets the GitHub Action create the new clean `main`.

**Step 1: Merge the current feature branch into main first**

```bash
# Make sure all lesson-02 work is on main before renaming
git checkout main
git merge feature/lesson-02-ip-fundamentals --no-ff -m "Merge feature/lesson-02-ip-fundamentals"
git push origin main
```

**Step 2: Create the instructor branch from main**

```bash
git checkout main
git checkout -b instructor
git push -u origin instructor
```

**Step 3: Rename default branch on GitHub**

```bash
# Change the default branch to instructor first (so main can be force-pushed)
gh repo edit --default-branch instructor
```

**Step 4: Delete and recreate main as the student view**

```bash
# Delete old main on remote
git push origin --delete main

# Create a clean main by running the sync logic locally
git checkout instructor
git checkout -b main-student

# Remove admin files
while IFS= read -r pattern; do
  [[ "$pattern" =~ ^[[:space:]]*#.*$ || -z "${pattern// }" ]] && continue
  find . -path "./$pattern" -o -path "./$pattern*" 2>/dev/null | while read f; do rm -rf "$f"; done
  git ls-files "$pattern" 2>/dev/null | while read f; do rm -rf "$f"; done
done < .admin-files
rm -rf .github/workflows/sync-student-branch.yml
rm -f .admin-files
find . -type d -empty -not -path './.git/*' -delete 2>/dev/null || true

git add -A
git commit -m "sync: initial student branch from instructor"
git push -u origin main-student:main

# Clean up local temp branch
git checkout instructor
git branch -D main-student
```

**Step 5: Set default branch back to main**

```bash
gh repo edit --default-branch main
```

**Step 6: Verify**

```bash
# Check that main exists and is clean
git fetch origin
git log --oneline origin/main -3
git log --oneline origin/instructor -3

# Verify admin files are NOT on main
git show origin/main:CLAUDE.md 2>&1 | head -1
# Expected: "fatal: path 'CLAUDE.md' does not exist"

# Verify admin files ARE on instructor
git show origin/instructor:CLAUDE.md 2>&1 | head -1
# Expected: "# CLAUDE.md - Project Instructions for Claude Code"
```

**Step 7: Trigger a test sync**

Make a trivial change on instructor and push to verify the Action runs:

```bash
git checkout instructor
echo "" >> .admin-files
git add .admin-files
git commit -m "chore: test sync action"
git push origin instructor
```

Check GitHub Actions tab to verify the workflow ran and main was updated.

---

### Task 5: Update local setup

**Step 1: Set local default branch tracking**

```bash
git checkout instructor
git branch --set-upstream-to=origin/instructor
```

**Step 2: Delete local main branch (optional, reduces confusion)**

```bash
git branch -D main
```

**Step 3: Verify you're working from instructor**

```bash
git branch -vv
# Should show: * instructor -> origin/instructor
```
