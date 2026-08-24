---
title: Create a work item
description: Create a work item via Plane API. HTTP request format, parameters, scopes, and example responses for create a work item.
keywords: plane, plane api, rest api, api integration, issue, create a work item
---

# Create a work item

<div class="api-endpoint-badge">
  <span class="method post">POST</span>
  <span class="path">/api/v1/workspaces/{workspace_slug}/projects/{project_id}/work-items/</span>
</div>

<div class="api-two-column">
<div class="api-left">

Create a new work item in the specified project with the provided details.

> **Creation defaults (fork extension).** This fork fills two fields when the request body
> does not carry them: an absent `assignees` assigns the authenticated caller, and an absent
> `target_date` becomes today. **An absent field and an explicitly empty one are not the
> same thing** — `"assignees": []` and `"target_date": null` are honoured as deliberate
> choices and are left alone. See [Creation defaults](#creation-defaults) below for the full
> contract. Not part of the upstream Plane API.


<div class="params-section">

### Path Parameters

<div class="params-list">

<ApiParam name="project_id" type="string" :required="true">

The unique identifier of the project.

</ApiParam>

<ApiParam name="workspace_slug" type="string" :required="true">

The workspace_slug represents the unique workspace identifier for a workspace in Plane. It can be found in the URL. For example, in the URL `https://app.plane.so/my-team/projects/`, the workspace slug is `my-team`.

</ApiParam>

</div>
</div>

<div class="params-section">

### Body Parameters

<div class="params-list">

<ApiParam name="assignees" type="array" :required="false">

Assignees. **Omit the key** to have the work item assigned to the authenticated caller;
send `[]` to create it deliberately unassigned. See [Creation defaults](#creation-defaults).

</ApiParam>

<ApiParam name="labels" type="array" :required="false">

Labels.

</ApiParam>

<ApiParam name="type_id" type="string" :required="false">

Type id.

</ApiParam>

<ApiParam name="parent" type="string" :required="false">

Parent.

</ApiParam>

<ApiParam name="deleted_at" type="string" :required="false">

Deleted at.

</ApiParam>

<ApiParam name="point" type="integer" :required="false">

Point.

</ApiParam>

<ApiParam name="name" type="string" :required="true">

Name.

</ApiParam>

<ApiParam name="description_html" type="string" :required="false">

Description html.

</ApiParam>

<ApiParam name="description_stripped" type="string" :required="false">

Description stripped.

</ApiParam>

<ApiParam name="priority" type="string" :required="false">

- `urgent` - Urgent
- `high` - High
- `medium` - Medium
- `low` - Low
- `none` - None

</ApiParam>

<ApiParam name="start_date" type="string" :required="false">

Start date.

</ApiParam>

<ApiParam name="target_date" type="string" :required="false">

Target date. **Omit the key** to have it set to today in the caller's own timezone; send
`null` to create the work item deliberately without a due date. See
[Creation defaults](#creation-defaults).

</ApiParam>

<ApiParam name="sequence_id" type="integer" :required="false">

Sequence id.

</ApiParam>

<ApiParam name="sort_order" type="number" :required="false">

Sort order.

</ApiParam>

<ApiParam name="completed_at" type="string" :required="false">

Completed at.

</ApiParam>

<ApiParam name="archived_at" type="string" :required="false">

Archived at.

</ApiParam>

<ApiParam name="last_activity_at" type="string" :required="false">

Last activity at.

</ApiParam>

<ApiParam name="is_draft" type="boolean" :required="false">

Is draft.

</ApiParam>

<ApiParam name="external_source" type="string" :required="false">

External source.

</ApiParam>

<ApiParam name="external_id" type="string" :required="false">

External id.

</ApiParam>

<ApiParam name="created_by" type="string" :required="false">

Created by.

</ApiParam>

<ApiParam name="state" type="string" :required="false">

State.

</ApiParam>

<ApiParam name="estimate_point" type="string" :required="false">

Estimate point.

</ApiParam>

<ApiParam name="type" type="string" :required="false">

Type.

</ApiParam>

</div>
</div>

<div class="params-section">

### Scopes

`projects.work_items:write`

</div>

<div class="params-section">

### Creation defaults

Fork extension; not part of the upstream Plane API.

The server fills two fields when the request body does not carry them:

- **No `assignees` key** — the work item is assigned to the authenticated caller.
- **No `target_date` key** — the due date is set to today.

#### Absent is not the same as empty

This is the only part of the behaviour a client can get wrong silently, because there is no
error either way:

| Request body | Result |
| --- | --- |
| no `assignees` key | assigned to the caller |
| `"assignees": []` | left unassigned — a deliberate choice, honoured |
| `"assignees": ["<uuid>"]` | exactly that, unchanged |
| no `target_date` key | today, in the caller's timezone |
| `"target_date": null` | no due date — a deliberate choice, honoured |
| `"target_date": "2026-12-25"` | exactly that, unchanged |

A client that initialises unset optional fields to `[]` or `null` before serialising will opt
every one of its users out of the defaults. If you are using an SDK, check whether it strips
null-valued keys before the request: a serialiser that drops them (Python's
`model_dump(exclude_none=True)`) makes `null` unreachable, so `"target_date": null` cannot be
expressed on create and the opt-out has to be a follow-up `PATCH`. One that preserves them
(`JSON.stringify`) lets both intents through.

#### Assignee precedence

1. The project's own `default_assignee`, when set and still an active project member at
   `role >= 15`. Unchanged from before this feature, and it applies **even when `assignees`
   is an empty list**.
2. Otherwise the authenticated caller — but only when the `assignees` key was absent.
3. If neither is an active project member at `role >= 15`, the work item is created
   unassigned rather than assigned to a user who cannot access it.

#### Due-date resolution

`target_date` defaults to **today in the authenticated caller's own timezone**
(`user_timezone` on their profile), not the server's UTC date. For a caller at UTC+7 the two
disagree for the first seven hours of their day.

When `start_date` is set and later than that date, the default is `start_date` instead. This
guarantees the default can never trigger the endpoint's own `Start date cannot exceed target
date` validation — a request carrying a future `start_date` and no `target_date` succeeded
before this feature and still succeeds.

#### Scope

Updates never default: a `PATCH` clearing either field leaves it cleared. Intake creation is
excluded and receives neither default. Drafts, sub-work items and epics are included.

</div>

</div>

<div class="api-right">

<CodePanel title="Create a work item" :languages="['cURL', 'Python', 'JavaScript']">
<template #curl>

```bash
curl -X POST \
  "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/work-items/" \
  -H "X-API-Key: $PLANE_API_KEY" \
  # Or use -H "Authorization: Bearer $PLANE_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
  "name": "Example Name",
  "description": "Example description",
  "priority": "medium",
  "state": "550e8400-e29b-41d4-a716-446655440000",
  "assignees": [
    "550e8400-e29b-41d4-a716-446655440000"
  ],
  "labels": [
    "550e8400-e29b-41d4-a716-446655440000"
  ],
  "external_id": "550e8400-e29b-41d4-a716-446655440000",
  "external_source": "github"
}'
```

</template>
<template #python>

```python
import requests

response = requests.post(
    "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/work-items/",
    headers={"X-API-Key": "your-api-key"},
    json={
      "name": "Example Name",
      "description": "Example description",
      "priority": "medium",
      "state": "550e8400-e29b-41d4-a716-446655440000",
      "assignees": [
"550e8400-e29b-41d4-a716-446655440000"
      ],
      "labels": [
"550e8400-e29b-41d4-a716-446655440000"
      ],
      "external_id": "550e8400-e29b-41d4-a716-446655440000",
      "external_source": "github"
    }
)
print(response.json())
```

</template>
<template #javascript>

```javascript
const response = await fetch("https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/work-items/", {
  method: "POST",
  headers: {
    "X-API-Key": "your-api-key",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    name: "Example Name",
    description: "Example description",
    priority: "medium",
    state: "550e8400-e29b-41d4-a716-446655440000",
    assignees: ["550e8400-e29b-41d4-a716-446655440000"],
    labels: ["550e8400-e29b-41d4-a716-446655440000"],
    external_id: "550e8400-e29b-41d4-a716-446655440000",
    external_source: "github",
  }),
});
const data = await response.json();
```

</template>
</CodePanel>

<ResponsePanel status="201">

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Example Name",
  "description": "Example description",
  "sequence_id": 1,
  "priority": "high",
  "assignees": ["550e8400-e29b-41d4-a716-446655440000"],
  "labels": ["550e8400-e29b-41d4-a716-446655440000"],
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
```

</ResponsePanel>

</div>

</div>
