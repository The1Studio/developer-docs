---
title: Preview a module cascade
description: Preview what a module cascade would touch via the Plane API. HTTP request format, parameters, scopes, and example responses, including the over-cap and empty-module shapes.
keywords: plane, plane api, rest api, api integration, module, cascade, preview, terminal status, completed, cancelled
---

# Preview a module cascade

<div class="api-endpoint-badge">
  <span class="method get">GET</span>
  <span class="path">/api/cascade-ext/workspaces/{workspace_slug}/projects/{project_id}/modules/{module_id}/cascade-preview/</span>
</div>

<div class="api-two-column">
<div class="api-left">

Read-only. Answers "what would cascade if this module's status moved to `status` right now" —
call this **before** the module's status actually changes, to decide whether to prompt for
confirmation and what to show in it. See
[How the item set is built](/api-reference/module-cascade/overview#how-the-item-set-is-built) and
[Eligibility](/api-reference/module-cascade/overview#eligibility) for what `items`, `depth`,
`is_module_member`, and `reason` mean.

<div class="params-section">

### Path Parameters

<div class="params-list">

<ApiParam name="module_id" type="string" :required="true">

The unique identifier of the module.

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

### Query Parameters

<div class="params-list">

<ApiParam name="status" type="string" :required="true">

The **module** status to preview cascading to. One of `completed` or `cancelled`. This is a
module status, not a work item state group — it is a different query parameter from the `group`
used by the per-work-item cascade preview endpoint.

</ApiParam>

</div>
</div>

<div class="params-section">

### Scopes

API key authentication or an OAuth token with equivalent access. Available to workspace/project
admins, members, and guests — the same read access as viewing the module itself.

</div>

</div>

<div class="api-right">

<CodePanel title="Preview a module cascade" :languages="['cURL', 'Python', 'JavaScript']">
<template #curl>

```bash
curl -X GET \
  "https://api.plane.so/api/cascade-ext/workspaces/my-workspace/projects/project-uuid/modules/module-uuid/cascade-preview/?status=completed" \
  -H "X-API-Key: $PLANE_API_KEY" \
  # Or use -H "Authorization: Bearer $PLANE_OAUTH_TOKEN"
```

</template>
<template #python>

```python
import requests

response = requests.get(
    "https://api.plane.so/api/cascade-ext/workspaces/my-workspace/projects/project-uuid/modules/module-uuid/cascade-preview/",
    headers={"X-API-Key": "your-api-key"},
    params={"status": "completed"},
)
print(response.json())
```

</template>
<template #javascript>

```javascript
const response = await fetch(
  "https://api.plane.so/api/cascade-ext/workspaces/my-workspace/projects/project-uuid/modules/module-uuid/cascade-preview/?status=completed",
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

<ResponsePanel status="200" title="TYPICAL RESPONSE">

```json
{
  "target_group": "completed",
  "depth_capped": false,
  "over_cap": false,
  "cap": 100,
  "summary": {
    "total_live": 3,
    "eligible": 2,
    "ineligible": 1,
    "already_terminal": 1
  },
  "items": [
    {
      "id": "4af68566-94a4-4eb3-94aa-50dc9427067b",
      "identifier": "PLANE-42",
      "name": "Ship the cascade preview endpoint",
      "depth": 0,
      "is_module_member": true,
      "project_id": "6436c4ae-fba7-45dc-ad4a-5440e17cb1b2",
      "project_name": "Plane",
      "state_id": "7c1e2b3a-1234-4eb3-94aa-50dc9427abcd",
      "state_name": "In Progress",
      "state_group": "started",
      "target_state_id": "b69b19ae-261f-428c-899f-dd58efaa36c0",
      "eligible": true,
      "reason": null
    },
    {
      "id": "9e3f2b1a-5678-4eb3-94aa-50dc9427cdef",
      "identifier": "PLANE-43",
      "name": "Write the endpoint tests",
      "depth": 1,
      "is_module_member": false,
      "project_id": "6436c4ae-fba7-45dc-ad4a-5440e17cb1b2",
      "project_name": "Plane",
      "state_id": "d4e5f6a7-89ab-4cde-8f12-345678901234",
      "state_name": "Todo",
      "state_group": "unstarted",
      "target_state_id": "b69b19ae-261f-428c-899f-dd58efaa36c0",
      "eligible": true,
      "reason": null
    },
    {
      "id": "1a2b3c4d-5e6f-4789-abcd-ef0123456789",
      "identifier": "OPS-7",
      "name": "Rotate the staging API key",
      "depth": 0,
      "is_module_member": true,
      "project_id": "8b2c4d5e-90ab-4cde-8f12-345678901234",
      "project_name": "Ops",
      "state_id": "2b3c4d5e-6f78-4901-bcde-f01234567890",
      "state_name": "Backlog",
      "state_group": "backlog",
      "target_state_id": null,
      "eligible": false,
      "reason": "no_matching_state"
    }
  ]
}
```

</ResponsePanel>

<ResponsePanel status="200" title="EMPTY MODULE — NO LIVE MEMBERS">

```json
{
  "target_group": "completed",
  "depth_capped": false,
  "over_cap": false,
  "cap": 100,
  "summary": {
    "total_live": 0,
    "eligible": 0,
    "ineligible": 0,
    "already_terminal": 0
  },
  "items": []
}
```

</ResponsePanel>

<ResponsePanel status="200" title="OVER THE CAP">

```json
{
  "target_group": "completed",
  "depth_capped": false,
  "over_cap": true,
  "cap": 100,
  "summary": {
    "total_live": 214,
    "eligible": 190,
    "ineligible": 24,
    "already_terminal": 8
  },
  "items": []
}
```

</ResponsePanel>

<ResponsePanel status="400" title="INVALID status">

```json
{
  "error": "status must be one of completed|cancelled"
}
```

</ResponsePanel>

<ResponsePanel status="400" title="ARCHIVED MODULE">

```json
{
  "error": "module is archived"
}
```

</ResponsePanel>

<ResponsePanel status="404">

```json
{
  "error": "module not found"
}
```

</ResponsePanel>

</div>

</div>

## Notes

- `target_group` echoes the work-item state group the module `status` maps to. For `completed`
  and `cancelled` this is the identical string to the requested `status` — the two are the same
  vocabulary here, but the field is named `target_group` because it is a work item state group,
  not a module status.
- `items` never includes a terminal (pruned) node — `summary.already_terminal` is the only place
  that count shows up. See [How the item set is
  built](/api-reference/module-cascade/overview#how-the-item-set-is-built).
- When `over_cap` is `true`, `items` is empty even though `summary.total_live` reports the real
  count — the list is refused, not truncated. See [The
  cap](/api-reference/module-cascade/overview#the-cap).
- This call changes nothing. It is safe to call repeatedly, e.g. to refresh a confirmation dialog.
