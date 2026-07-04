---
title: Get a work item's estimate
description: Get a work item's workload estimate via Plane API. HTTP request format, parameters, scopes, and example responses, including parent-issue rollup shape.
keywords: plane, plane api, rest api, api integration, workload, estimate, get work item estimate, rollup
---

# Get a work item's estimate

<div class="api-endpoint-badge">
  <span class="method get">GET</span>
  <span class="path">/api/v1/workspaces/{workspace_slug}/projects/{project_id}/issues/{issue_id}/workload-estimate/</span>
</div>

<div class="api-two-column">
<div class="api-left">

Retrieve the hour estimate for a single work item.

If the work item is a **parent** (it has one or more countable sub-items — see
[Countable work items](/api-reference/workload/overview#countable-work-items)), `hours` is always
`null`, `is_parent` is `true`, and a computed `rollup` object is included instead. See
[Parent issues cannot be estimated directly](/api-reference/workload/overview#parent-issues-cannot-be-estimated-directly).

<div class="params-section">

### Path Parameters

<div class="params-list">

<ApiParam name="issue_id" type="string" :required="true">

The unique identifier of the work item.

</ApiParam>

<ApiParam name="project_id" type="string" :required="true">

The unique identifier of the project.

</ApiParam>

<ApiParam name="workspace_slug" type="string" :required="true">

The workspace_slug represents the unique workspace identifier for a workspace in Plane. It can be found in the URL. For example, in the URL `https://app.plane.so/my-team/projects/`, the workspace slug is `my-team`.

</ApiParam>

</div>
</div>

<div class="params-section">

### Scopes

API key authentication or an OAuth token with equivalent access.

</div>

</div>

<div class="api-right">

<CodePanel title="Get a work item's estimate" :languages="['cURL', 'Python', 'JavaScript']">
<template #curl>

```bash
curl -X GET \
  "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/issues/issue-uuid/workload-estimate/" \
  -H "X-API-Key: $PLANE_API_KEY" \
  # Or use -H "Authorization: Bearer $PLANE_OAUTH_TOKEN"
```

</template>
<template #python>

```python
import requests

response = requests.get(
    "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/issues/issue-uuid/workload-estimate/",
    headers={"X-API-Key": "your-api-key"}
)
print(response.json())
```

</template>
<template #javascript>

```javascript
const response = await fetch(
  "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/issues/issue-uuid/workload-estimate/",
  {
    method: "GET",
    headers: {
      "X-API-Key": "your-api-key",
    },
  }
);
const data = await response.json();
```

</template>
</CodePanel>

<ResponsePanel status="200" title="LEAF WORK ITEM, NO ESTIMATE SET">

```json
{
  "hours": null,
  "is_parent": false
}
```

</ResponsePanel>

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

<ResponsePanel status="200" title="PARENT WORK ITEM (HAS SUB-ITEMS)">

```json
{
  "hours": null,
  "is_parent": true,
  "rollup": {
    "hours": 10.0,
    "done_hours": 6.0,
    "percent": 0.6,
    "due_date": "2026-08-12",
    "leaf_count": 2
  }
}
```

</ResponsePanel>

<ResponsePanel status="403">

```json
{
  "error": "You are not allowed to view this estimate"
}
```

</ResponsePanel>

</div>

</div>

## Notes

- `id`, `issue`, `created_at`, and `updated_at` are only present when a stored estimate row
  exists for the work item. A leaf with no estimate set returns just `{"hours": null, "is_parent": false}`.
  A parent that still has a legacy stored row from before it gained sub-items may include these
  fields alongside `hours: null` — the row is kept but ignored, never surfaced as a usable value.
- The `403` response above applies only to a workspace guest whose access is restricted to their
  own assigned work items, when the requested work item is not assigned to them.
