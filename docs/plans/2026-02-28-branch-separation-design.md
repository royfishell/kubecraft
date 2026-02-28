# Student/Instructor Branch Separation -- Design

## Overview

Separate admin/instructor content from student-facing content using two branches and a GitHub Action that automatically syncs.

## Branch Model

- **`instructor`** -- working branch with all content (student + admin)
- **`main`** -- student-facing only, auto-generated from instructor minus admin files
- Students fork `main` and never see admin content

## Admin-Only Files (stripped from main)

| Pattern | Description |
|---------|-------------|
| `**/script.md` | Video recording scripts |
| `lessons/clab/COURSE_PLAN.md` | Course-level planning |
| `lessons/clab/VIDEO_SCRIPT_TEMPLATE.md` | Script template |
| `CLAUDE.md` | AI assistant instructions |
| `docs/plans/**` | Design docs and implementation plans |

These patterns are listed in a `.admin-files` manifest on the instructor branch. The GitHub Action reads this manifest to know what to strip.

## GitHub Action

Triggered on push to `instructor`. Steps:
1. Checkout instructor branch
2. Read `.admin-files` manifest
3. Delete listed files/patterns
4. Force-push result to main

## Workflow

1. Work on `instructor` branch (or feature branches off `instructor`)
2. Push/merge to `instructor`
3. GitHub Action automatically syncs to `main` minus admin files
4. Students see clean view on `main`

## Migration

1. Rename current `main` to `instructor`
2. GitHub Action creates new clean `main`
3. Update CLAUDE.md git workflow to reference `instructor`
4. GitHub default branch stays `main` (student-facing)
