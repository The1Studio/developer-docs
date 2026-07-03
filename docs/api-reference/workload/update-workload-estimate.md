---
title: Update a work item's estimate
description: Update a work item's workload estimate via Plane API. HTTP request format, parameters, scopes, and example responses, including the 400 error returned for parent issues.
keywords: plane, plane api, rest api, api integration, workload, estimate, update work item estimate, parent has children
---

# Update a work item's estimate

<div class="api-endpoint-badge">
  <span class="method put">PUT</span>
  <span class="path">/api/v1/workspaces/{workspace_slug}/projects/{project_id}/issues/{issue_id}/workload-estimate/</span>
</div>

<div class="api-two-column">
<div class="api-left">

Set or replace the hour estimate for a work item. Creates the estimate if none exists yet, or
overwrites the existing value otherwise.

Rejected with `400 Bad Request` if the work item is a **parent** (it has one or more countable
sub-items) — see [Parent issues cannot be estimated directly](/api-reference/workload/overview#parent-issues-cannot-be-estimated-directly).
Set the estimate on the sub-items instead; the parent's total is computed automatically.

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

### Body Parameters

<div class="params-list">

<ApiParam name="hours" type="number" :required="true">

Estimated hours for the work item. Must be `0` or greater, and no more than `10000`. Quantized to
2 decimal places on write.

</ApiParam>

</div>
</div>

<div class="params-section">

### Scopes

API key authentication or an OAuth token with equivalent access.

</div>

</div>

<div class="api-right">

<CodePanel title="Update a work item's estimate" :languages="['cURL', 'Python', 'JavaScript']">
<template #curl>

```bash
curl -X PUT \
  "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/issues/issue-uuid/workload-estimate/" \
  -H "X-API-Key: $PLANE_API_KEY" \
  # Or use -H "Authorization: Bearer $PLANE_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "hours": 6.5
  }'
```

</template>
<template #python>

```python
import requests

response = requests.put(
    "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/issues/issue-uuid/workload-estimate/",
    headers={"X-API-Key": "your-api-key"},
    json={"hours": 6.5},
)
print(response.json())
```

</template>
<template #javascript>

```javascript
const response = await fetch(
  "https://api.plane.so/api/v1/workspaces/my-workspace/projects/project-uuid/issues/issue-uuid/workload-estimate/",
  {
    method: "PUT",
    headers: {
      "X-API-Key": "your-api-key",
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ hours: 6.5 }),
  }
);
const data = await response.json();
```

</template>
</CodePanel>

<ResponsePanel status="200">

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "issue": "4af68566-94a4-4eb3-94aa-50dc9427067b",
  "hours": 6.5,
  "created_at": "2026-06-20T10:30:00Z",
  "updated_at": "2026-06-21T09:15:00Z"
}
```

</ResponsePanel>

<ResponsePanel status="400" title="WORK ITEM HAS SUB-ITEMS">

```json
{
  "error": "This issue has sub-items — set estimates on the sub-items instead.",
  "error_code": "PARENT_HAS_CHILDREN"
}
```

</ResponsePanel>

<ResponsePanel status="404">

```json
{
  "error": "issue not found"
}
```

</ResponsePanel>

</div>

</div>

## Notes

- `error_code` is included specifically so that clients (SDKs, integrations) can branch on the
  `PARENT_HAS_CHILDREN` case programmatically instead of matching the human-readable `error`
  string.
- If a work item's sub-items are all later removed, cancelled, or deleted, it reverts to being a
  regular leaf and can be estimated again.
- `DELETE` on this same endpoint remains allowed even for a parent work item, to clear a legacy
  stored value — only `PUT` is blocked.
