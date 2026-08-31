---
title: Apply a module cascade
description: Apply a module cascade via the Plane API. HTTP request format, parameters, scopes, and example responses, including the rejected-reason enum and the over-cap 400 shape.
keywords: plane, plane api, rest api, api integration, module, cascade, apply, terminal status, completed, cancelled, item_ids
---

# Apply a module cascade

<div class="api-endpoint-badge">
  <span class="method post">POST</span>
  <span class="path">/api/cascade-ext/workspaces/{workspace_slug}/projects/{project_id}/modules/{module_id}/cascade-apply/</span>
</div>

<div class="api-two-column">
<div class="api-left">

Sets the module's `status` and, in the same transaction, moves a caller-selected subset of its
currently eligible live items into the matching state. A failure partway through rolls the
module's own status change back too — either everything in the transaction is written, or nothing
is.

`item_ids` is a **request, not an authorization**: eligibility is always re-derived from the
current state of the tree, exactly as [preview](/api-reference/module-cascade/preview-module-cascade)
computes it. An id you post that is not currently eligible does not get forced through — it comes
back in `rejected` with a reason instead. See
[The item_ids parameter](/api-reference/module-cascade/overview#the-item_ids-parameter).

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

### Body Parameters

<div class="params-list">

<ApiParam name="status" type="string" :required="true">

The module status to apply. One of `completed` or `cancelled`.

</ApiParam>

<ApiParam name="item_ids" type="array" :required="false">

Work item ids to cascade alongside the module's status change. Omitted or `null` cascades every
currently eligible item; an explicit `[]` cascades none — only the module's status moves.

</ApiParam>

</div>
</div>

<div class="params-section">

### Scopes

API key authentication or an OAuth token with equivalent access. Available to workspace/project
admins and members only — the same roles allowed to write a module.

</div>

</div>

<div class="api-right">

<CodePanel title="Apply a module cascade" :languages="['cURL', 'Python', 'JavaScript']">
<template #curl>

```bash
curl -X POST \
  "https://api.plane.so/api/cascade-ext/workspaces/my-workspace/projects/project-uuid/modules/module-uuid/cascade-apply/" \
  -H "X-API-Key: $PLANE_API_KEY" \
  # Or use -H "Authorization: Bearer $PLANE_OAUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
  "status": "completed",
  "item_ids": [
    "4af68566-94a4-4eb3-94aa-50dc9427067b",
    "9e3f2b1a-5678-4eb3-94aa-50dc9427cdef"
  ]
}'
```

</template>
<template #python>

```python
import requests

response = requests.post(
    "https://api.plane.so/api/cascade-ext/workspaces/my-workspace/projects/project-uuid/modules/module-uuid/cascade-apply/",
    headers={"X-API-Key": "your-api-key"},
    json={
        "status": "completed",
        "item_ids": [
            "4af68566-94a4-4eb3-94aa-50dc9427067b",
            "9e3f2b1a-5678-4eb3-94aa-50dc9427cdef",
        ],
    },
)
print(response.json())
```

</template>
<template #javascript>

```javascript
const response = await fetch(
  "https://api.plane.so/api/cascade-ext/workspaces/my-workspace/projects/project-uuid/modules/module-uuid/cascade-apply/",
  {
    method: "POST",
    headers: {
      "X-API-Key": "your-api-key",
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      status: "completed",
      item_ids: ["4af68566-94a4-4eb3-94aa-50dc9427067b", "9e3f2b1a-5678-4eb3-94aa-50dc9427cdef"],
    }),
  }
);
const data = await response.json();
```

</template>
</CodePanel>

<ResponsePanel status="200" title="EVERYTHING REQUESTED WAS ACCEPTED">

```json
{
  "module": "b69b19ae-261f-428c-899f-dd58efaa36c0",
  "status": "completed",
  "updated": ["4af68566-94a4-4eb3-94aa-50dc9427067b", "9e3f2b1a-5678-4eb3-94aa-50dc9427cdef"],
  "rejected": []
}
```

</ResponsePanel>

<ResponsePanel status="200" title="SOME IDS REJECTED">

```json
{
  "module": "b69b19ae-261f-428c-899f-dd58efaa36c0",
  "status": "completed",
  "updated": ["4af68566-94a4-4eb3-94aa-50dc9427067b"],
  "rejected": [
    { "id": "1a2b3c4d-5e6f-4789-abcd-ef0123456789", "reason": "no_matching_state" },
    { "id": "c3d4e5f6-7890-4abc-9def-012345678901", "reason": "under_terminal_ancestor" },
    { "id": "f1e2d3c4-b5a6-4978-8f01-234567890abc", "reason": "not_in_module_tree" }
  ]
}
```

</ResponsePanel>

<ResponsePanel status="400" title="OVER THE CAP">

```json
{
  "error": "cascade exceeds MAX_MODULE_CASCADE_ITEMS",
  "total_live": 214,
  "cap": 100
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

<ResponsePanel status="400" title="INVALID item_ids">

```json
{
  "error": "item_ids must be a list, null, or omitted"
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

- `updated` is the sorted list of ids that were actually moved — a subset of what you posted (or,
  when `item_ids` was omitted, of every currently eligible item).
- `rejected` entries carry every id from your request that did **not** move, each with a
  `reason`. See the full enum in [The item_ids
  parameter](/api-reference/module-cascade/overview#the-item_ids-parameter).
- On the over-the-cap `400`, **nothing is written** — not the cascaded items, and not the module's
  own `status`. `total_live` reports the real live count so the caller knows how far over the
  100-item `cap` it is. Retrying with a smaller, explicit `item_ids` list only helps once the
  underlying live count itself is back under the cap — the cap is checked against the module's
  full live tree, not against the size of `item_ids`.
- The module's own status change and every accepted item's state change happen in one
  transaction: a failure partway through rolls both back.
- Notifications: the module's own status change notifies watchers as usual. None of the cascaded
  item state changes fire watcher notifications — a 90-item cascade does not send 90
  notifications.
