# FlowSpec MCP Tools Reference (v4.0.0)

All 29 tools provided by the FlowSpec MCP server. Tool names are prefixed with `flowspec_` when called.

---

## Read Tools

### flowspec_list_projects

List all FlowSpec projects with names and dates.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| *(none)* | | | |

**Returns:** Markdown list of projects with name, ID (UUID), and last-updated timestamp.

---

### flowspec_get_json

Get the full JSON spec for a FlowSpec project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |

**Returns:** JSON string with spec data (dataPoints, components, transforms, tables, dataFlow, screens).

---

### flowspec_get_project

Get raw canvas_state JSON for a FlowSpec project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |

**Returns:** JSON string containing `{ nodes, edges, screens }`.

---

### flowspec_search_nodes

Search for nodes by label across all projects.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Search term to match against node labels |
| `nodeType` | enum | No | Filter: `datapoint`, `component`, or `transform` |

**Returns:** Markdown list of matches with label, nodeType, projectName, nodeId, projectId.

---

### flowspec_get_screen_context

Get screen/region/element structure for a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `screenId` | string | No | Specific screen ID (omit for all screens) |

**Returns:** JSON array of screen objects with regions and linked elements.

---

## Write Tools

### flowspec_create_project

Create a new FlowSpec project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Project name |
| `canvas_state` | object | No | Initial canvas state `{ nodes, edges, screens }`. Defaults to empty. |

**Returns:** `{ name, id }` of the created project.

---

### flowspec_update_project

Update a project name or replace its entire canvas state.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `name` | string | No | New project name |
| `canvas_state` | object | No | Full replacement canvas state JSON |

**Returns:** `{ name, id }` of the updated project.

---

### flowspec_delete_project

Delete a FlowSpec project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |

**Returns:** Confirmation with project ID.

---

### flowspec_clone_project

Clone a project with all its canvas state.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sourceProjectId` | string | Yes | UUID of the project to clone |
| `newName` | string | Yes | Name for the cloned project |

**Returns:** `{ name, id }` of the cloned project.

---

### flowspec_create_node

Add a node to a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `type` | enum | Yes | `datapoint`, `component`, `transform`, or `table` |
| `position` | `{x, y}` | Yes | Canvas position |
| `data` | object | Yes | Node data (varies by type — see below) |

**Node data by type:**

- **datapoint**: `{ label, type, source, sourceDefinition, constraints }`
- **component**: `{ label, displays, captures, wireframeRef? }`
- **transform**: `{ label, type, description, inputs, outputs, logic }`
- **table**: `{ label, sourceType, columns, endpoint? }`

**Returns:** Created node with generated UUID.

---

### flowspec_update_node

Update a node's data or position.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `nodeId` | string | Yes | UUID of the node |
| `data` | object | No | Data fields to merge into existing node data |
| `position` | `{x, y}` | No | New canvas position |

**Returns:** Updated node object.

---

### flowspec_delete_node

Remove a node and all its connected edges.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `nodeId` | string | Yes | UUID of the node |

**Returns:** Confirmation of deletion.

---

### flowspec_create_edge

Connect two nodes with an edge.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `source` | string | Yes | Source node ID |
| `target` | string | Yes | Target node ID |
| `sourceHandle` | enum/null | No | `source-left`, `source-top`, `source-right`, `source-bottom`, or `null` (auto) |
| `targetHandle` | enum/null | No | `target-left`, `target-top`, `target-right`, `target-bottom`, or `null` (auto) |
| `data` | object | No | Additional edge data |

> **Note:** All edges are created with type `flows-to`. Edge type is no longer configurable.

**Returns:** Created edge with UUID.

---

### flowspec_update_edge

Update an edge's label or connection handles.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `edgeId` | string | Yes | UUID of the edge |
| `label` | string | No | Edge label text |
| `sourceHandle` | enum/null | No | `source-left`, `source-top`, `source-right`, `source-bottom`, or `null` (auto) |
| `targetHandle` | enum/null | No | `target-left`, `target-top`, `target-right`, `target-bottom`, or `null` (auto) |
| `data` | object | No | Additional edge data to merge |

**Returns:** Updated edge object.

---

### flowspec_delete_edge

Remove an edge from a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `edgeId` | string | Yes | UUID of the edge |

**Returns:** Confirmation of deletion.

---

### flowspec_analyse_project

Run quality analysis on a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |

**Returns:** Analysis report with:
- Total/connected node counts
- Orphan nodes (no edges) with labels and types
- Duplicate labels with node IDs and types
- Screen references for orphan nodes

---

## Bulk / Import Tools

### flowspec_import_json

Import a spec into a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the target project |
| `json` | string | Yes | JSON spec string |
| `autoLayout` | boolean | No | Run dagre layout after import (default: `true`) |
| `layoutDirection` | enum | No | `TB` (default), `BT`, `LR`, `RL` |
| `merge` | boolean | No | `true` = add to existing, `false` = replace (default: `false`) |

**Returns:** Import statistics (datapoints, components, transforms, tables, edges, screens, skipped counts).

**Usage patterns:**
- Full rewrite: `merge: false` (default) — replaces entire canvas
- Incremental add: `merge: true` — adds new nodes/edges to existing canvas
- Round-trip edit: `get_json` -> modify -> `import_json(merge: false)`

---

### flowspec_auto_layout

Auto-arrange nodes using dagre hierarchical layout.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `direction` | enum | No | `TB` (default), `BT`, `LR`, `RL` |
| `pinnedNodeIds` | string[] | No | Node IDs to keep in current positions |

**Returns:** Repositioned node count, direction, and pinned count.

---

## Image / Screen Tools

### flowspec_upload_image

Upload an image file for use as a wireframe/screenshot.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `filePath` | string | Yes | Absolute path to image on disk |
| `width` | number | No | Image width (auto-detected if omitted) |
| `height` | number | No | Image height (auto-detected if omitted) |

**Returns:** `{ url, width, height, filename }`.

---

### flowspec_create_screen

Add a wireframe screen to a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `name` | string | Yes | Screen name (e.g. "Login Page") |
| `imageUrl` | string | No | URL of wireframe image (optional for skeleton screens) |
| `imageWidth` | number | No | Image width in pixels (default: 1920) |
| `imageHeight` | number | No | Image height in pixels (default: 1080) |
| `imageFilename` | string | No | Original filename |

**Returns:** `{ id, name }` of the created screen.

---

### flowspec_update_screen

Update a screen's name or image properties.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `screenId` | string | Yes | UUID of the screen |
| `name` | string | No | New screen name |
| `imageUrl` | string | No | New image URL |
| `imageWidth` | number | No | New width |
| `imageHeight` | number | No | New height |
| `imageFilename` | string | No | New filename |

**Returns:** Updated screen object.

---

### flowspec_delete_screen

Delete a screen and all its regions.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `screenId` | string | Yes | UUID of the screen |

**Returns:** Confirmation of deletion.

---

### flowspec_add_region

Add an annotated region to a screen.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `screenId` | string | Yes | UUID of the screen |
| `label` | string | No | Region label (e.g. "Login Form") |
| `position` | `{x, y}` | Yes | Top-left corner, percentage 0-100 |
| `size` | `{width, height}` | Yes | Region size, percentage 0-100 |
| `elementIds` | string[] | No | Canvas node IDs linked to this region |
| `componentNodeId` | string | No | Component node ID (for promoted regions) |

**Returns:** Created region with UUID.

---

### flowspec_update_region

Update a region's label, position, size, or linked elements.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `screenId` | string | Yes | UUID of the screen |
| `regionId` | string | Yes | UUID of the region |
| `label` | string | No | New label |
| `position` | `{x, y}` | No | New position (percentage 0-100) |
| `size` | `{width, height}` | No | New size (percentage 0-100) |
| `elementIds` | string[] | No | New linked node IDs |
| `componentNodeId` | string | No | New component node ID |

**Returns:** Updated region object.

---

### flowspec_remove_region

Remove a region from a screen.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `screenId` | string | Yes | UUID of the screen |
| `regionId` | string | Yes | UUID of the region |

**Returns:** Confirmation of removal.

---

## Decision Tree Tools (v4.0.0)

### flowspec_list_decision_trees

List all decision trees for a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |

**Returns:** Markdown list of trees with name, ID, source node label, trace depth, and last-updated timestamp.

---

### flowspec_get_decision_tree

Get a decision tree with full node/edge structure.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `treeId` | string | Yes | ID of the decision tree |

**Returns:** Tree summary (name, description, source, trace depth) plus full JSON structure of decision/condition/outcome nodes and edges.

---

### flowspec_delete_decision_tree

Delete a decision tree from a project.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `treeId` | string | Yes | ID of the decision tree |

**Returns:** Confirmation of deletion.

---

### flowspec_analyse_decision_tree

Analyse a decision tree's structure and quality.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `projectId` | string | Yes | UUID of the project |
| `treeId` | string | Yes | ID of the decision tree |

**Returns:** Analysis report with:
- Summary: total nodes, edges, max depth, leaf node count
- Node type distribution
- Outcome distribution (approve/reject/escalate/etc.)
- Issues: orphan nodes, under-branched decisions (<2 branches), non-outcome leaf nodes

---

> See also: `examples/lesson.md` in the main FlowSpec repo for common MCP workflow patterns and gotchas.
