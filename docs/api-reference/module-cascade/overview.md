---
title: Overview
description: Plane Module Cascade API overview. Learn how moving a module to completed or cancelled can cascade that terminal status onto its live work items via REST API.
keywords: plane, plane api, rest api, api integration, module, cascade, terminal status, bulk update, completed, cancelled
---

# Overview

Module cascade lets you move a module to a terminal status (`completed` or `cancelled`) and, in
the same call, move a caller-selected subset of its **live** work items — direct members and
their live descendants — into a matching state. A preview endpoint answers "what would this
cascade touch right now" before you commit to it.

> This is a Plane fork extension and is not part of the upstream Plane API. It is available on
> self-hosted instances that run this fork.

Both endpoints are mounted **outside** `/api/v1`, at `/api/cascade-ext/`:

```
GET  /api/cascade-ext/workspaces/{workspace_slug}/projects/{project_id}/modules/{module_id}/cascade-preview/
POST /api/cascade-ext/workspaces/{workspace_slug}/projects/{project_id}/modules/{module_id}/cascade-apply/
```

A parallel pair of endpoints exists for cascading a single work item's own terminal-status change
onto its sub-items (`.../issues/{issue_id}/cascade-preview/` and `.../cascade-apply/`); this page
covers the **module**-level pair only. The two share the same underlying walk and the same
`reason` values below.

## How the item set is built

Starting from every **live** work item currently assigned to the module (the module's direct
members), the walk follows sub-items level by level, going up to 20 levels deep. A work item
already in a terminal state group (`completed` or `cancelled`) is **pruned**: it is excluded from
the result, and — this is the important part — its own sub-items are never visited, listed, or
touched, even if they are still live. A terminal node represents a decision already made about
that branch; the walk does not reach past it.

- `depth: 0` — a direct module member.
- `depth: N` (N ≥ 1) — a descendant N levels below one.
- `is_module_member` — emitted explicitly on every item, and is **not** the same thing as
  `depth == 0`: a work item can be both a direct module member and a descendant of another module
  member, in which case it still carries `is_module_member: true` at whatever depth the walk found
  it.

## Eligibility

Every live item the walk finds is either **eligible** (it will move if included in the apply
call) or not, and every ineligible item carries a `reason`:

| `reason`            | Meaning                                                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `no_matching_state` | The item's project has no state whose group matches the target terminal group, so there is nowhere to move it. `target_state_id` is `null` for these items. |
| `no_permission`     | You are not an active member of the item's project.                                                                                                         |

An eligible item's `reason` is `null`.

## The cap

A single cascade will not write more than `MAX_MODULE_CASCADE_ITEMS` (**100**) live items. This is
checked against the live count **before** anything is written — it is a refusal, not a
truncation:

- **Preview** — when the live count exceeds the cap, `over_cap` is `true` and `items` is an
  **empty array**, even though `summary.total_live` still reports the real count. The list is
  deliberately not truncated to the first 100 — a truncated list would silently under-report what
  an apply call would actually touch.
- **Apply** — over the cap, the whole call is rejected with `400` and **nothing is written**,
  including the module's own status change. See [Apply a module
  cascade](/api-reference/module-cascade/apply-module-cascade) for the exact error shape.

## The item_ids parameter

The apply endpoint takes an optional `item_ids` list. It is a **request, not an authorization** —
eligibility is always re-derived server-side from the current state of the tree, so posting an id
that is not currently eligible does not force it through; it comes back in `rejected` instead.

- `item_ids` omitted or `null` — every currently eligible item cascades.
- `item_ids: []` — nothing cascades; only the module's own status changes.

An id you post that is **not** accepted lands in `rejected` with one of these reasons (in addition
to the two above):

| `reason`                  | Meaning                                                                                                                                                                                                                                                          |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `already_terminal`        | The id is itself a terminal node the walk encountered — a module member or descendant already sitting in the target terminal group (or the other one). It is exactly what preview prunes from `items`.                                                           |
| `under_terminal_ancestor` | The id is a genuine, live descendant, but the walk never reached it because an ancestor between it and the module is already terminal and pruned that branch. This distinguishes a real (but pruned) descendant from one that was never part of the tree at all. |
| `not_in_module_tree`      | The id is neither a module member nor a descendant of one — it is genuinely outside this cascade.                                                                                                                                                                |
| `not_eligible`            | A defensive fallback that should not normally appear — every ineligible item already carries `no_matching_state` or `no_permission`.                                                                                                                             |

## Archived modules

Both endpoints reject an archived module with `400 {"error": "module is archived"}`, mirroring the
core module endpoints' own refusal to write an archived module.

## Permissions

- **Preview** — workspace/project admins, members, and guests: the same read access as viewing
  the module itself. Preview changes nothing.
- **Apply** — workspace/project admins and members only, the same roles allowed to write a module.

## Endpoints

- [Preview a module cascade](/api-reference/module-cascade/preview-module-cascade)
- [Apply a module cascade](/api-reference/module-cascade/apply-module-cascade)
