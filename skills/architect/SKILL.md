---
name: architect
description: >-
  Create and manage FlowSpec data architecture projects using the MCP server.
  Three workflows: overlay wireframes with dataflows, map an existing codebase
  onto the canvas, or design new wireframe screens. Requires the FlowSpec
  desktop app or MCP server (flowspec_* tools).
argument_hint: "[wireframe images | codebase path | 'new project']"
---

# FlowSpec Architect (MCP)

Creates FlowSpec projects — the reverse direction of `spec`/`wireframe`. Instead of *consuming* a FlowSpec YAML to build code, this skill *produces* FlowSpec projects from wireframes, codebases, or design sessions.

**Core pattern:** Generate YAML spec -> `flowspec_import_yaml` -> `flowspec_get_yaml` -> refine -> re-import.

**Requires:** `flowspec_*` MCP tools (FlowSpec desktop app or standalone MCP server).

---

## Setup: Verify Tools & Select Project

1. Call `flowspec_list_projects` to confirm the MCP server is connected.
   - If the call fails, the MCP server is not running. Tell the user to open the FlowSpec desktop app or start the MCP server.
2. List existing projects — ask the user to select one or create a new project.
3. Before any destructive rewrite, offer to `flowspec_clone_project` as a safety backup.

---

## Workflow Router

Determine which workflow to follow based on user input:

| User provides | Workflow |
|---------------|----------|
| Wireframe images (PNG/JPG/screenshots) | **UC1**: Wireframe -> Dataflow |
| A codebase path to analyse | **UC2**: Codebase -> Dataflow |
| "Design screens" / "new project" | **UC3**: Design New Screens |
| A `.yaml` file | Direct import via `flowspec_import_yaml` |

---

## UC1: Wireframe -> Dataflow

### Step 1: Upload wireframe images
For each wireframe image file the user provides:
```
flowspec_upload_image(filePath: "/path/to/wireframe.png")
```
Returns `{ url, width, height, filename }`.

### Step 2: Create screens
For each uploaded image:
```
flowspec_create_screen(projectId, name: "Login Page", imageUrl: url, imageWidth: width, imageHeight: height, imageFilename: filename)
```
Returns `{ id, name }` — save the screen ID for region linking.

### Step 3: Analyse wireframe images
Use Claude vision to examine each wireframe image:
- Identify distinct UI regions (header, sidebar, forms, data displays, navigation)
- List data elements visible in each region (form fields, labels, values, tables)
- Note interactive elements (buttons, dropdowns, toggles)
- Identify data relationships (which fields feed into which displays)

### Step 4: Interview user
Ask the user about:
- Data types for each identified element (string, number, boolean, etc.)
- Data sources — what is user-entered (captured) vs computed/fetched (inferred)?
- Business logic — what calculations, validations, or workflows exist?
- Data persistence — which database tables or APIs back these data elements?
- Constraints — required fields, enums, ranges, formats

### Step 5: Build YAML spec
Assemble the complete YAML following the [v1.2.0 schema](references/yaml-schema.md):
- `dataPoints` — one per identified data element
- `components` — one per page/screen, listing `displays` and `captures`
- `transforms` — one per business logic function
- `tables` — one per database table or API endpoint
- `dataFlow` — edges connecting all nodes
- `screens` — include screen IDs from Step 2 (leave `regions` empty for now)

### Step 6: Import YAML
```
flowspec_import_yaml(projectId, yaml: yamlString, autoLayout: true, layoutDirection: "TB")
```

### Step 7: Add regions
For each UI region identified in Step 3, link it to the corresponding screen:
```
flowspec_add_region(projectId, screenId, label: "Login Form", position: {x, y}, size: {width, height}, elementIds: [nodeId1, nodeId2])
```
Use percentage coordinates (0-100) relative to the wireframe image.

### Step 8: Auto-layout
```
flowspec_auto_layout(projectId, direction: "TB")
```

### Step 9: Quality check
```
flowspec_analyse_project(projectId)
```
Review orphan nodes and duplicate labels. Fix any issues.

### Step 10: Iterate
```
flowspec_get_yaml(projectId)
```
Review the exported YAML. Refine and re-import as needed:
```
flowspec_import_yaml(projectId, yaml: refinedYaml, merge: false)
```

---

## UC2: Codebase -> Dataflow

This workflow analyses an existing codebase and produces a FlowSpec data architecture map. See [codebase-analysis.md](references/codebase-analysis.md) for the full 8-step pipeline.

### Step 1: Scope assessment
- Count routes/pages, utility modules, database tables/API endpoints
- Detect framework (SvelteKit, Next.js, Express, Remix, Nuxt)
- Decide granularity (single YAML vs per-domain split)

### Step 2: Check existing specs
```
flowspec_list_projects
flowspec_search_nodes(query: "relevant-term")
```
If a project already exists for this codebase, retrieve it with `flowspec_get_yaml` and update rather than recreate.

### Step 3: Discover DataPoints
Scan the codebase for data elements:
- Type definitions and interfaces -> entity fields
- Server loaders/handlers -> SQL queries, API responses
- Form components -> captured DataPoints
- Calculation outputs -> inferred DataPoints

### Step 4: Discover Components
One Component per page route. For each, determine:
- `displays`: which DataPoint IDs it renders
- `captures`: which DataPoint IDs it accepts as input

### Step 5: Discover Transforms
Scan utility/service modules for business logic functions:
- `formula` — calculations, aggregations
- `validation` — data checking, constraint enforcement
- `workflow` — multi-step processes, API integrations

### Step 6: Map DataFlow edges
Trace data paths through the codebase:
- What does each server loader query? (DataPoint -> Component)
- What does each form submit? (Component -> DataPoint)
- What utility functions process data? (Transform edges)

### Step 7: Build YAML spec
Assemble using [v1.2.0 schema](references/yaml-schema.md).

### Step 8: Create project and import
```
flowspec_create_project(name: "My App Architecture")
flowspec_import_yaml(projectId, yaml: yamlString)
```

### Step 9: Layout and analyse
```
flowspec_auto_layout(projectId, direction: "TB")
flowspec_analyse_project(projectId)
```

### Step 10: (Optional) Overlay screenshots
If the user has screenshots of the running app:
```
flowspec_upload_image(filePath: "/path/to/screenshot.png")
flowspec_create_screen(projectId, name: "Dashboard", imageUrl: url, ...)
flowspec_add_region(projectId, screenId, label: "KPI Cards", position: {...}, size: {...}, elementIds: [...])
```

---

## UC3: Design New Screens

### Step 1: Obtain screen image
The user creates a design in an external tool (Figma, Pencil MCP, etc.) and provides a screenshot, or starts from an empty screen.

### Step 2: Upload and create screen
Same as UC1 Steps 1-2 (upload image, create screen).

### Step 3: Define regions collaboratively
Walk through the screenshot with the user. For each UI zone:
- Name the region
- Identify percentage coordinates (position + size)
- List the data elements it contains

### Step 4: Build or extend YAML
Create new `dataPoints`, `components`, and `transforms` for the screen's data, or reference existing nodes from the project.

### Step 5: Import with merge
```
flowspec_import_yaml(projectId, yaml: newYaml, merge: true)
```
`merge: true` adds new nodes/edges without replacing existing ones.

### Step 6: Link regions
```
flowspec_add_region(projectId, screenId, ...)
```

---

## The YAML Round-Trip

The core editing loop for FlowSpec projects:

```
flowspec_get_yaml(projectId)     # Export current state
  -> edit YAML locally           # Add/modify/remove nodes and edges
flowspec_import_yaml(projectId, yaml, merge: false)  # Re-import
```

### Key parameters
- `merge: false` (default) — replaces entire canvas. Use for rewrites.
- `merge: true` — adds new nodes/edges to existing canvas. Use for incremental additions.
- `autoLayout: true` (default) — runs dagre layout after import.
- `layoutDirection` — `TB` (top-to-bottom, default), `LR` (left-to-right, good for data pipelines), `BT`, `RL`.

### ID stability
When round-tripping, reuse node/edge IDs from `flowspec_get_yaml` to preserve identity. New nodes get new IDs; existing nodes keep their original IDs.

---

## Quality Checklist

Before finalising a project, verify:

- [ ] Every `captured` DataPoint is captured by at least one Component
- [ ] Every `inferred` DataPoint has a Transform producing it
- [ ] Every Transform has at least one input and one output
- [ ] Every Component either displays or captures at least one DataPoint
- [ ] No orphan nodes (verify with `flowspec_analyse_project`)
- [ ] No duplicate labels (verify with `flowspec_analyse_project`)
- [ ] Consistent ID naming: `dp-*`, `comp-*`, `tx-*`, `tbl-*`
- [ ] Screens have regions covering all relevant components
- [ ] Edge types follow conventions:
  - `flows-to` for data movement
  - `derives-from` for Transform -> inferred DataPoint
  - `transforms` for DataPoint -> Transform
  - `validates` for validation Transform -> DataPoint
  - `contains` for Table -> DataPoint

---

## Tool Quick Reference

| Category | Tool | Purpose |
|----------|------|---------|
| **Read** | `flowspec_list_projects` | List all projects |
| | `flowspec_get_yaml` | Export project as YAML |
| | `flowspec_get_project` | Raw canvas JSON |
| | `flowspec_search_nodes` | Search nodes by label |
| | `flowspec_get_screen_context` | Screen/region structure |
| **Project** | `flowspec_create_project` | Create new project |
| | `flowspec_update_project` | Update name or canvas |
| | `flowspec_delete_project` | Delete project |
| | `flowspec_clone_project` | Clone project (backup) |
| **Node** | `flowspec_create_node` | Add a node |
| | `flowspec_update_node` | Update node data/position |
| | `flowspec_delete_node` | Remove node + edges |
| **Edge** | `flowspec_create_edge` | Connect two nodes |
| | `flowspec_update_edge` | Update edge type/handles |
| | `flowspec_delete_edge` | Remove edge |
| **Bulk** | `flowspec_import_yaml` | Import YAML spec |
| | `flowspec_auto_layout` | Auto-arrange nodes |
| | `flowspec_analyse_project` | Orphan/duplicate analysis |
| **Screen** | `flowspec_upload_image` | Upload wireframe image |
| | `flowspec_create_screen` | Add screen to project |
| | `flowspec_update_screen` | Update screen properties |
| | `flowspec_delete_screen` | Remove screen + regions |
| | `flowspec_add_region` | Annotate screen region |
| | `flowspec_update_region` | Update region |
| | `flowspec_remove_region` | Remove region |

See [mcp-tools.md](references/mcp-tools.md) for full parameter details on all 25 tools.
