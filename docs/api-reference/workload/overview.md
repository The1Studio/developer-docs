---
title: Overview
description: Plane Workload API overview. Learn about per-issue hour estimates, workload matrix, and parent-issue progress rollups via REST API.
keywords: plane, plane api, rest api, api integration, workload, estimates, hours, rollup
---

# Overview

Workload lets you record a time estimate (in hours) on a work item and read back aggregated
workload data — a day/week/month matrix per assignee, and, for a work item that has sub-items,
a computed rollup of hours, completion percentage, and due date across its sub-tree.

> This is a Plane fork extension and is not part of the upstream Plane API. It is available on
> self-hosted instances that run this fork.

<div class="api-two-column">
<div class="api-left">

## The Workload Estimate Object

### Attributes

- `id` _uuid_

  Unique identifier for the estimate row. Omitted when the issue has no stored estimate.

- `issue` _uuid_

  ID of the work item the estimate belongs to. Omitted when the issue has no stored estimate.

- `hours` _number_ or _null_

  Estimated hours for the work item (0–10000, quantized to 2 decimal places). Always `null` for
  a parent issue (an issue with sub-items) — see [Parent issues](#parent-issues-cannot-be-estimated-directly)
  below.

- `created_at`, `updated_at` _timestamp_

  Timestamps when the estimate was created and last updated. Omitted when the issue has no
  stored estimate.

- `is_parent` _boolean_

  Whether the work item currently has one or more countable sub-items (see
  [Countable work items](#countable-work-items)). When `true`, `hours` is always `null` and a
  `rollup` object is included.

</div>
<div class="api-right">

<ResponsePanel status="200" title="LEAF WORK ITEM, WITH AN ESTIMATE">

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "issue": "4af68566-94a4-4eb3-94aa-50dc9427067b",
  "hours": 6.5,
  "created_at": "2026-06-20T10:30:00Z",
  "updated_at": "2026-06-21T09:15:00Z",
  "is_parent": false
}
```

</ResponsePanel>

</div>
</div>

<div class="api-two-column">
<div class="api-left">

## The Rollup Object

Returned as the `rollup` field on a parent issue's estimate (see
[Get a work item's estimate](/api-reference/workload/get-workload-estimate)) and as the value
type of the [bulk rollups](/api-reference/workload/list-workload-rollups) endpoint.

### Attributes

- `hours` _number_

  Sum of estimated hours across the parent's countable **leaf** descendants (descendants with no
  countable children of their own). Only leaves with a stored estimate greater than `0` count.

- `done_hours` _number_

  Subset of `hours` contributed by leaves whose state group is `completed`.

- `percent` _number_ or _null_

  `done_hours / hours`, rounded to 4 decimal places. `null` when `hours` is `0` (no estimated
  leaves), so a client can distinguish "no data" from "0% done".

- `due_date` _string_ or _null_

  ISO `YYYY-MM-DD` date — the latest `target_date` across **all** countable descendants (not just
  leaves). This intentionally differs from `hours`, which only sums leaves: an intermediate
  sub-item's due date still counts even though its own hours don't.

- `leaf_count` _integer_

  Number of countable leaf descendants that have a stored estimate greater than `0`.

</div>
<div class="api-right">

<ResponsePanel status="200" title="THE ROLLUP OBJECT">

```json
{
  "hours": 10.0,
  "done_hours": 6.0,
  "percent": 0.6,
  "due_date": "2026-08-12",
  "leaf_count": 2
}
```

</ResponsePanel>

</div>
</div>

## Countable work items

A descendant counts toward a rollup — and toward whether a work item is treated as a "parent" —
only if it is **countable**:

- not soft-deleted (`deleted_at` is null)
- not archived (`archived_at` is null)
- not a draft (`is_draft` is `false`)
- its state group is **not** `cancelled` and **not** `triage` (a work item with no state, i.e. a
  null state, still counts)

A non-countable descendant (for example a cancelled sub-item) prunes its entire subtree from the
rollup — its own children are not counted either, even if they would otherwise be countable.
Recursion covers the full sub-item tree, up to 10 levels deep from the requested work item.

A work item is a **parent** when it has one or more countable direct or indirect children. If all
of a work item's children are cancelled, deleted, archived, drafts, or in triage, the work item
reverts to being a regular, directly-estimable leaf.

## Parent issues cannot be estimated directly

Once a work item has at least one countable sub-item, its own hour estimate is no longer editable
— hours are expected to live on the sub-items instead, and the parent's progress is derived from
them:

- `GET` on a parent's estimate returns `hours: null`, `is_parent: true`, and a computed `rollup`
  object (see [Get a work item's estimate](/api-reference/workload/get-workload-estimate)).
- `PUT` on a parent's estimate is rejected with `400 Bad Request` and
  `error_code: "PARENT_HAS_CHILDREN"` (see
  [Update a work item's estimate](/api-reference/workload/update-workload-estimate)).
- `DELETE` on a parent's estimate is still allowed, to clean up a legacy value left over from
  before the work item gained sub-items. A legacy stored value on a parent is otherwise ignored
  everywhere — it is never returned as `hours` and never included in the workload matrix or the
  bulk estimates endpoint (see below).

## Behavior changes on existing endpoints

Two previously-shipped workload endpoints changed their result set to stay consistent with the
"parents don't carry their own hours" rule above. Their request/response shape is otherwise
unchanged.

- **Workload matrix** — `GET /api/v1/workspaces/{workspace_slug}/workload/` and
  `GET /api/v1/workspaces/{workspace_slug}/projects/{project_id}/workload/` now aggregate hours
  from **leaf** work items only. A work item that has sub-items no longer contributes its own
  (legacy) estimate to the matrix, even if one is still stored for it — this avoids double-counting
  hours that also show up under its sub-items.
- **Bulk estimates** — `GET /api/v1/workspaces/{workspace_slug}/workload-estimates/?issue_ids=...`
  now omits rows for any id that is a parent. A parent's stored estimate is never returned by this
  endpoint, the same way a single-item `GET` returns `null` for it — a parent looks identical to
  "no estimate set". A leaf's stored estimate is still returned even if the leaf's own state is
  `cancelled` — only the matrix applies a state-group filter, this endpoint does not.

## Restricted-guest visibility

A guest whose access is restricted to their assigned work items sees a **scope-partial** rollup:
descendants outside their visible scope are silently excluded from `hours`, `done_hours`,
`percent`, `due_date`, and `leaf_count`. This is by design, not a bug — it mirrors how every other
workload endpoint scopes data down for restricted guests. A restricted guest's rollup for the same
parent work item can therefore be numerically smaller than what an admin or member sees.

## Known limitation — stale after adding or removing sub-items

The Plane web app does not automatically refetch a rollup after a sub-item is added to or removed
from a work item; the displayed `Σ hours · percent` stays stale until the page is reloaded. Fetch
the estimate or rollup endpoint again if you need the up-to-date value immediately after such a
change. This is a known v1 limitation, not an API contract — the API itself always returns
current data.

## Endpoints

- [Get a work item's estimate](/api-reference/workload/get-workload-estimate)
- [Update a work item's estimate](/api-reference/workload/update-workload-estimate)
- [List rollups for multiple work items](/api-reference/workload/list-workload-rollups)
