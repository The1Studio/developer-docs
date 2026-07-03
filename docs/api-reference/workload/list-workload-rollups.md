---
title: List rollups for multiple work items
description: List workload rollups for multiple work items via Plane API. HTTP request format, parameters, scopes, and example responses for the bulk workload rollups endpoint.
keywords: plane, plane api, rest api, api integration, workload, rollup, bulk, sub-items, progress
---

# List rollups for multiple work items

<div class="api-endpoint-badge">
  <span class="method get">GET</span>
  <span class="path">/api/v1/workspaces/{workspace_slug}/workload-rollups/</span>
</div>

<div class="api-two-column">
<div class="api-left">

Retrieve computed [rollups](/api-reference/workload/overview#the-rollup-object) — total hours,
completed hours, percent done, due date, and leaf count — for a batch of work item ids in one
call.

Only ids that are currently a **parent** (have one or more countable sub-items) are included in
the response. Ids that are not a parent, and ids outside the caller's project access, are both
simply absent from the response body — there is no way to distinguish "not a parent" from "not
visible to you" from the response alone; this matches the existing bulk estimates endpoint.

This endpoint is workspace-scoped — it has no `project_id` in the path, and a single request can
mix ids from any project the caller can access within the workspace.

<div class="params-section">

### Path Parameters

<div class="params-list">

<ApiParam name="workspace_slug" type="string" :required="true">

The workspace_slug represents the unique workspace identifier for a workspace in Plane. It can be found in the URL. For example, in the URL `https://app.plane.so/my-team/projects/`, the workspace slug is `my-team`.

</ApiParam>

</div>
</div>

<div class="params-section">

### Query Parameters

<div class="params-list">

<ApiParam name="issue_ids" type="string" :required="true">

Comma-separated list of work item ids to compute rollups for. Must contain at least 1 id, and no
more than 500.

</ApiParam>

</div>
</div>

<div class="params-section">

### Scopes

API key authentication or an OAuth token with equivalent access.

</div>

</div>

<div class="api-right">

<CodePanel title="List rollups for multiple work items" :languages="['cURL', 'Python', 'JavaScript']">
<template #curl>

```bash
curl -X GET \
  "https://api.plane.so/api/v1/workspaces/my-workspace/workload-rollups/?issue_ids=4af68566-94a4-4eb3-94aa-50dc9427067b,7c1e2b3a-1234-4eb3-94aa-50dc9427abcd" \
  -H "X-API-Key: $PLANE_API_KEY" \
  # Or use -H "Authorization: Bearer $PLANE_OAUTH_TOKEN"
```

</template>
<template #python>

```python
import requests

response = requests.get(
    "https://api.plane.so/api/v1/workspaces/my-workspace/workload-rollups/",
    headers={"X-API-Key": "your-api-key"},
    params={
        "issue_ids": "4af68566-94a4-4eb3-94aa-50dc9427067b,7c1e2b3a-1234-4eb3-94aa-50dc9427abcd"
    },
)
print(response.json())
```

</template>
<template #javascript>

```javascript
const response = await fetch(
  "https://api.plane.so/api/v1/workspaces/my-workspace/workload-rollups/?issue_ids=4af68566-94a4-4eb3-94aa-50dc9427067b,7c1e2b3a-1234-4eb3-94aa-50dc9427abcd",
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

<ResponsePanel status="200">

```json
{
  "4af68566-94a4-4eb3-94aa-50dc9427067b": {
    "hours": 10.0,
    "done_hours": 6.0,
    "percent": 0.6,
    "due_date": "2026-08-12",
    "leaf_count": 2
  }
}
```

</ResponsePanel>

<ResponsePanel status="200" title="NONE OF THE REQUESTED IDS ARE PARENTS">

```json
{}
```

</ResponsePanel>

<ResponsePanel status="400" title="EMPTY issue_ids">

```json
{
  "error": "issue_ids must not be empty"
}
```

</ResponsePanel>

<ResponsePanel status="400" title="TOO MANY issue_ids">

```json
{
  "error": "Too many issue_ids (max 500, got 501)"
}
```

</ResponsePanel>

</div>

</div>

## Notes

- Authorization mirrors the bulk estimates endpoint
  (`GET /api/v1/workspaces/{workspace_slug}/workload-estimates/`) exactly: it is
  workspace-level (`WORKSPACE` permission scope, not per-project), and a workspace guest whose
  access is restricted to their own assigned work items receives a scope-partial result — see
  [Restricted-guest visibility](/api-reference/workload/overview#restricted-guest-visibility).
- Requesting only leaf (non-parent) ids is not an error — it returns `200 {}`, distinct from the
  `400` returned for an empty `issue_ids` parameter.
- In the unlikely case that the underlying sub-item tree for the requested ids is too large to
  compute in one request, the endpoint returns `400` with
  `{"error": "Result too large — narrow the issue_ids list."}`. Split the request into smaller
  batches of `issue_ids` if you hit this.
