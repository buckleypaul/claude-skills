---
name: doo:tasks
description: Use ONLY when the user explicitly asks to read, create, or manage items
  in their personal doo task list. NOT for tracking AI work or session progress.
---

# Doo Task Management

This skill provides access to the user's personal `doo` task list.

**SCOPE**: Only use when the user EXPLICITLY asks to interact with their doo task list.
Do NOT use this skill to track your own work or session progress.

## Permission Model

- **Read operations** (list, show): execute directly, no confirmation needed
- **Write operations** (add, edit, complete, delete, move): ALWAYS present what you intend
  to create/modify and wait for explicit user confirmation before executing any command

## Reading Tasks

```bash
doo task list --json                          # all active tasks
doo task list --status backlog --json         # filter by status
doo task list --tag backend --json            # filter by tag
doo task list --search "keyword" --json       # search titles
doo task list --done --json                   # completed tasks
doo task show <ID> --json                     # single task detail
```

## JSON Output Schema

```json
{
  "tasks": [{
    "id": "uuid",
    "title": "string",
    "priority": 0,
    "tags": ["string"],
    "dueDate": "2026-03-10",
    "status": "untriaged",
    "description": "string or null",
    "notes": "string or null",
    "subtasks": [{"id": "uuid", "title": "string", "completed": false}]
  }]
}
```

- `priority`: 0 = highest, 1 = medium, 2 = lowest (default)
- `dueDate`: `yyyy-MM-dd` string or `null`
- `status`: `untriaged` | `backlog` | `in_progress` | `in_review`

## Proposing Tasks (before writing)

Before creating tasks, present a numbered list for user review:

> I'll add the following tasks to your doo list:
> 1. `"Update API docs !1 #backend @next-monday %backlog"`
> 2. `"Review PR from Jane #review !2"`
>
> Shall I add these? (confirm / edit / cancel)

## Writing Tasks (after confirmation)

```bash
# Add — inline syntax
doo task add "title !N #tag @date %status /description"

# Complete / uncomplete
doo task complete <ID>
doo task uncomplete <ID>

# Edit
doo task edit <ID> --priority 1 --tag new --remove-tag old --due tomorrow --status inreview

# Delete
doo task delete <ID>

# Move pipeline status
doo task move <ID> backlog        # untriaged | backlog | inprogress | inreview

# Subtasks
doo task subtask add <taskID> "subtask title"
doo task subtask complete <taskID> <subtaskID>
doo task subtask delete <taskID> <subtaskID>
```

## Inline Syntax Reference

```
title text [!0-2] [#tag …] [@today|tomorrow|yyyy-MM-dd] [%status] [/description]
```

- `!0` = highest priority, `!1` = medium, `!2` = lowest (default, can omit)
- `#tag` — one or more tags (repeat for multiple)
- `@today`, `@tomorrow`, `@yyyy-MM-dd` — due date
- `%untriaged`, `%backlog`, `%inprogress`, `%inreview` (also accepts hyphens/underscores)
- `/description text` — freeform description, must come last

Example: `Fix login bug !1 #backend @tomorrow %backlog /check token expiry`

## Task IDs

IDs shown in `doo task list` output are either:
- Row numbers: `1`, `2`, `3` (positional, change as tasks are added/removed)
- 8-char UUID prefix: `a3f2b1c4` (stable, from `--json` output `id` field)

Prefer UUID prefixes for reliability in multi-step workflows.

## Common Workflow: Meeting Notes → Tasks

1. Read the meeting notes or ask the user to paste them
2. Extract action items assigned to the user
3. For each item determine: title, priority (`!N`), tags (`#tag`), due date (`@date`), status (`%status`)
4. Present the full proposed list using inline syntax for user confirmation
5. On approval, run `doo task add` for each item sequentially
6. Report the created task IDs (use `doo task list --json` after adding to confirm)
