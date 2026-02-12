---
name: annotate
description: >-
  Use when indexing a codebase to extract FlowSpec-compatible elements
  (datapoints, components, transforms, tables, images, screens, data flows)
  from source files. Produces a persistent index with incremental re-indexing
  so only new or modified files are re-processed on subsequent runs.
argument_hint: "[path/to/directory]"
---

# FlowSpec Codebase Annotator

Index a codebase and extract FlowSpec-compatible elements (datapoints, components, transforms, tables, screens, data flows) from source files. Produces a persistent index with per-file timestamps so only new or modified files are re-processed on subsequent runs.

## Invocation

```
/flowspec:annotate [path]
```

- `path` (optional) — directory to index. Defaults to the current working directory.

## Behaviour

1. **Load or create the index file** at `<project-root>/.flowspec/codebase-index.json`.
2. **Discover source files** — walk the target directory for indexable files (see File Selection below).
3. **Diff against the index** — for each discovered file, compare its filesystem `mtime` to the `lastIndexed` timestamp stored in the index. Skip files whose `mtime` is older than or equal to `lastIndexed`.
4. **Analyse changed files** — read each new/modified file and extract FlowSpec elements (see Extraction Rules below).
5. **Update the index** — merge new extractions into the index, remove entries for deleted files, and rebuild the aggregated spec.
6. **Write the index** — save `.flowspec/codebase-index.json`.
7. **Report** — summarise what changed: files scanned, files skipped (unchanged), elements found, and the path to the index file.

## File Selection

### Include

Scan files matching these patterns:

```
**/*.ts  **/*.tsx  **/*.js  **/*.jsx
**/*.svelte  **/*.vue  **/*.astro
**/*.py  **/*.go  **/*.rs  **/*.java  **/*.kt
**/*.sql  **/*.prisma  **/*.graphql  **/*.gql
**/*.json (only package.json, tsconfig.json, schema files)
**/*.yaml  **/*.yml (only schema/config files)
```

### Exclude

Always skip:

```
node_modules/    .git/           dist/         build/
.svelte-kit/     .next/          .nuxt/        .output/
__pycache__/     target/         vendor/
*.min.js         *.min.css       *.map
*.lock           *.log           *.png  *.jpg  *.svg  *.ico  *.woff*
.env*            *.pem           *.key
```

Also skip any paths listed in `.gitignore` (if present).

## Index File Format

The index is stored at `.flowspec/codebase-index.json`:

```jsonc
{
  "version": "1.0.0",
  "createdAt": "2026-02-12T10:00:00.000Z",
  "lastRunAt": "2026-02-12T12:30:00.000Z",
  "projectRoot": "/absolute/path/to/project",
  "config": {
    "include": ["**/*.ts", "**/*.svelte"],
    "exclude": ["node_modules/**"]
  },
  "files": {
    "src/routes/login/+page.svelte": {
      "lastIndexed": "2026-02-12T12:30:00.000Z",
      "fileModified": "2026-02-12T11:45:00.000Z",
      "sizeBytes": 2340,
      "elements": {
        "dataPoints": [ /* ... */ ],
        "components": [ /* ... */ ],
        "transforms": [ /* ... */ ],
        "tables": [ /* ... */ ],
        "images": [ /* ... */ ],
        "dataFlow": [ /* ... */ ],
        "screens": [ /* ... */ ]
      }
    }
    // ... one entry per indexed file
  },
  "spec": {
    // Aggregated FlowSpec JSON (v1.2.0) — ready for import or MCP consumption
    // See "Aggregated Spec" section below
  }
}
```

### Per-File `elements` Object

Each file entry contains the FlowSpec elements extracted from that file. IDs are deterministic and scoped to the file to avoid collisions:

**ID format:** `idx:<relative-path>:<element-type>:<ordinal>` (or `idx:<workspace>:<relative-path>:<element-type>:<ordinal>` in monorepos — see Edge Cases below)
Example: `idx:src/routes/login/+page.svelte:dp:0`

This ensures IDs are stable across re-indexes (same file + same element = same ID) and globally unique across the project.

## Aggregated Spec

The top-level `spec` key contains a merged FlowSpec JSON document (v1.2.0 format) assembled from all file entries. This is the primary output — it can be:

- Imported directly into FlowSpec via MCP: `flowspec_create_project` with the spec
- Appended to an existing project via individual `flowspec_create_node` / `flowspec_create_edge` calls
- Read by any tool that understands FlowSpec JSON

```json
{
  "version": "1.2.0",
  "metadata": {
    "projectName": "<directory-name> (indexed)",
    "exportedAt": "2026-02-12T12:30:00.000Z",
    "nodeCount": 42,
    "edgeCount": 31,
    "sourceFiles": 15,
    "indexVersion": "1.0.0"
  },
  "dataPoints": [],
  "components": [],
  "transforms": [],
  "tables": [],
  "images": [],
  "dataFlow": [],
  "screens": []
}
```

## Extraction Rules

When reading a source file, identify and categorise the following elements into FlowSpec node types. Use the file's purpose, exports, and structure to determine what belongs where.

### DataPoints

A DataPoint represents a discrete piece of data that flows through the application. Extract these from:

| Source Pattern | DataPoint Properties |
|---|---|
| Form input fields (`<input>`, `<select>`, `bind:value`) | `source: "captured"`, type from input type |
| State variables (`$state`, `useState`, `ref()`, `let x = ...` in reactive context) | `source: "captured"` if user-set, `"inferred"` if computed |
| Props / component parameters (`$props()`, `export let`, function params on components) | `source: "captured"`, type from TypeScript annotation |
| API response fields (destructured fetch results, return types) | `source: "inferred"`, `sourceDefinition: "API: <endpoint>"` |
| Database column values (query results, ORM model fields) | `source: "inferred"`, `sourceDefinition: "DB: <table>.<column>"` |
| Environment variables (`process.env.*`, `$env/*`) | `source: "inferred"`, `sourceDefinition: "env"` |
| URL parameters / search params | `source: "captured"`, `sourceDefinition: "URL param"` |

> **Security — `.env` files:**
> Index variable **names** only — never index values, secrets, or credentials. The `.env*` exclusion pattern above already prevents reading `.env` files. Exception: `.env.example` and `.env.template` may be read for variable names only (they should not contain real secrets). Recommend adding `.flowspec/` to `.gitignore` so the index (which contains file paths and element names) is not committed.

**DataPoint JSON:**
```json
{
  "id": "idx:src/routes/login/+page.svelte:dp:0",
  "label": "User Email",
  "type": "string",
  "source": "captured",
  "sourceDefinition": "Text input on login form",
  "constraints": ["required", "email format"],
  "locations": [
    { "component": "idx:src/routes/login/+page.svelte:comp:0", "role": "input" }
  ]
}
```

**Type mapping:**
- `string` — text inputs, string variables, enum-like values
- `number` — numeric inputs, counters, IDs, amounts
- `boolean` — checkboxes, toggles, flags, boolean state
- `object` — complex objects, nested data, JSON blobs
- `array` — lists, collections, multi-select values

**Constraints** — include when determinable:
- `"required"` — non-optional, validated as present
- `"unique"` — must be unique (DB constraints, unique validation)
- `"email format"`, `"URL format"`, `"phone format"` — format validation
- `"min: N"`, `"max: N"`, `"minLength: N"`, `"maxLength: N"` — range/length
- `"enum: [values]"` — restricted set of values
- `"readonly"` — not user-modifiable (computed, derived)

**Constraint detection algorithm** — extract constraints from these sources (check all that apply to the file's language):

1. **TypeScript types:** `?` property → omit `"required"`; union literal types (e.g., `"a" | "b"`) → `"enum: [a, b]"`; `Readonly<T>` or `as const` → `"readonly"`
2. **Zod schemas:** `.min(N)` → `"min: N"`; `.max(N)` → `"max: N"`; `.email()` → `"email format"`; `.url()` → `"URL format"`; `.optional()` → omit `"required"`; `.regex(pattern)` → `"pattern: <pattern>"`
3. **HTML5 attributes:** `required` → `"required"`; `minlength="N"` → `"minLength: N"`; `maxlength="N"` → `"maxLength: N"`; `type="email"` → `"email format"`; `type="url"` → `"URL format"`; `pattern="..."` → `"pattern: ..."`; `min="N"` / `max="N"` → `"min: N"` / `"max: N"`
4. **Framework validators:** React Hook Form `rules: { required, minLength, ... }`, Svelte `use:validate` actions, Formik/Yup schema chains — map their constraint names to the same FlowSpec constraint strings above
5. **Database schema:** `NOT NULL` → `"required"`; `UNIQUE` → `"unique"`; `CHECK(...)` → extract the condition; Prisma `@unique` / `@default` / Drizzle `.notNull()` / `.unique()` — same mapping

**Merge rule:** Union all constraints found across sources. If two sources specify conflicting numeric ranges (e.g., HTML `min="0"` and Zod `.min(1)`), prefer the stricter (larger min, smaller max) value.

**`locations` format:** Each entry in the `locations` array has `{ "component": "<Component node ID>", "role": "input" | "output" }`. `"input"` means the component captures this DataPoint (it appears in the component's `captures` list); `"output"` means the component displays it (it appears in `displays`). Populated during Step 5.9, not during per-file extraction.

### Components

A Component represents a UI element or page that displays and/or captures data. Extract from:

| Source Pattern | Component |
|---|---|
| Svelte/Vue/React component files (`.svelte`, `.vue`, `.tsx`) | One component per file |
| Exported page components (`+page.svelte`, `page.tsx`, `index.vue`) | Component with `wireframeRef` = route path |
| Layout components (`+layout.svelte`, `layout.tsx`) | Component wrapping child components |
| Reusable UI components (form groups, cards, modals, tables) | Component with displays/captures from props |

**Extracting children** — 6-step algorithm:

1. **Parse imports:** Collect all component imports — named (`import { Button } from`), default (`import Button from`), aliased (`import { Button as Btn }`), and namespace (`import * as Icons from`). Identify which resolve to component files (`.svelte`, `.vue`, `.tsx`, `.jsx`) vs utility modules.
2. **Search template for usage:** For each imported component, search the template/JSX for actual usage. Match by tag name: `<Button`, `<Btn`, `<Icons.Home`, `{#each ... as}` wrappers, conditional renders (`{#if}`, `? ... :`). A component that is imported but never appears in the template is **not** a child.
3. **Handle dynamic imports:** `React.lazy(() => import('./Heavy'))`, Svelte `{#await import('./Heavy') then mod}`, Vue `defineAsyncComponent(() => import('./Heavy'))` — resolve the path, treat the result as a child if it appears in the template.
4. **Handle slot-based children:** When a component defines `<slot>` (Svelte), `{children}` (React), or `<slot>` (Vue), the children are provided by the **parent**, not the slot-defining component. Do not list slot-injected content as children of the component defining the slot.
5. **Order by first template appearance:** Sort children by the line/character offset of their first usage in the template. This preserves visual order.
6. **Build `children` array:** Each child entry is the child component's index ID (`idx:<path>:comp:<ordinal>`). During per-file extraction, if the child component hasn't been indexed yet, use a placeholder ID based on the import path — resolve to the real ID during aggregation (Step 5).

**Extracting layout:** Inspect the component's outermost template wrapper:
| Source Pattern | Layout |
|---|---|
| `display: flex` / `flex` / Tailwind `flex` class | `type: "flex"`, check `flex-direction` or `flex-col`/`flex-row` for direction |
| `display: grid` / Tailwind `grid` class | `type: "grid"`, check `grid-template-areas` for named areas |
| Sidebar + main pattern (two children, one narrow) | `type: "sidebar"` |
| Tab/tabbed interface (`<Tabs>`, role="tablist") | `type: "tabs"` |
| Vertical stack of sections with no explicit layout | `type: "stack"`, `direction: "column"` |
| No detectable structure | `type: "free"` |

**Extracting sizeHint:** Count displays + captures for `fieldCount`. Count `children` array length for `childCount`. Apply category thresholds.

**Component JSON:**
```json
{
  "id": "idx:src/routes/login/+page.svelte:comp:0",
  "label": "Login Page",
  "wireframeRef": "/login",
  "displays": ["idx:src/routes/login/+page.svelte:dp:2"],
  "captures": ["idx:src/routes/login/+page.svelte:dp:0", "idx:src/routes/login/+page.svelte:dp:1"],
  "children": ["idx:src/lib/components/LoginForm.svelte:comp:0", "idx:src/lib/components/OAuthButtons.svelte:comp:0"],
  "layout": {
    "type": "flex",
    "direction": "column",
    "areas": ["header", "main", "footer"]
  },
  "sizeHint": {
    "fieldCount": 3,
    "childCount": 2,
    "category": "medium"
  }
}
```

- `displays` — IDs of DataPoints this component renders (read-only display)
- `captures` — IDs of DataPoints this component captures from user input
- `wireframeRef` — the route path or meaningful reference string
- `children` — IDs of child Components this component imports and renders in its template. Establishes the nesting hierarchy (page → section → form → input).
- `layout` — how child elements are arranged:
  - `type` — `"flex"`, `"grid"`, `"stack"`, `"sidebar"`, `"tabs"`, or `"free"` (inferred from CSS classes, Tailwind utilities, or wrapper elements)
  - `direction` — `"row"` or `"column"` (for flex/stack)
  - `areas` — named layout regions if detectable (e.g., grid-template-areas, slot names, semantic sections like header/main/footer)
- `sizeHint` — data to help the canvas estimate node dimensions:
  - `fieldCount` — total displays + captures (how much content the component shows)
  - `childCount` — number of children
  - `category` — `"small"` (0-2 fields, 0 children), `"medium"` (3-6 fields or 1-3 children), `"large"` (7+ fields or 4+ children)

### Transforms

A Transform represents business logic, data processing, or validation. Extract from:

| Source Pattern | Transform |
|---|---|
| Validation functions (Zod schemas, `validate()`, form validators) | `type: "validation"` |
| Computed / derived values (`$derived`, `useMemo`, computed properties) | `type: "formula"` |
| Data transformation functions (mappers, formatters, parsers) | `type: "formula"` |
| API route handlers (`+server.ts`, `api/*.ts`, controllers) | `type: "workflow"` |
| Authentication logic (login, session checks, guards) | `type: "workflow"` |
| Database queries with logic (joins, aggregations, filters) | `type: "formula"` |
| Multi-step processes (checkout flows, wizards, state machines) | `type: "workflow"`, logic type `"steps"` |

**Transform JSON:**
```json
{
  "id": "idx:src/lib/auth.ts:tx:0",
  "type": "validation",
  "description": "Validate login credentials against database",
  "inputs": ["idx:src/routes/login/+page.svelte:dp:0", "idx:src/routes/login/+page.svelte:dp:1"],
  "outputs": ["idx:src/lib/auth.ts:dp:0"],
  "logic": {
    "type": "steps",
    "content": "1. Validate email format\n2. Query user by email\n3. Compare password hash\n4. Generate session token"
  }
}
```

### Tables

A Table represents a persistent data source. Extract from:

| Source Pattern | Table |
|---|---|
| Database schema definitions (SQL `CREATE TABLE`, Prisma `model`, Drizzle schema) | `sourceType: "database"` |
| API endpoint definitions (REST routes, GraphQL types) | `sourceType: "api"`, `endpoint` = URL |
| Static data files (JSON configs, CSV, seed data) | `sourceType: "file"` |
| In-memory stores (global stores, context providers with initial data) | `sourceType: "manual"` |

**Table JSON:**
```json
{
  "id": "idx:src/lib/server/db/schema.sql:tbl:0",
  "label": "users",
  "sourceType": "database",
  "columns": [
    { "name": "id", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "password_hash", "type": "string" },
    { "name": "created_at", "type": "string" }
  ],
  "endpoint": "postgres://neon/users"
}
```

### Images

An Image represents a static visual asset referenced in the codebase. Extract from:

| Source Pattern | Image |
|---|---|
| Static imports (`import logo from './logo.png'`) | `url` = import path |
| HTML `<img>` / `<Image>` tags with `src` attribute | `url` = src value |
| CSS/Tailwind background images (`background-image: url(...)`, `bg-[url(...)]`) | `url` = extracted URL |
| Public asset references (`/images/hero.png`, `/static/logo.svg`) | `url` = public path |
| Svelte `{@html}` or framework image components (`<Image>`, `<Picture>`, `next/image`) | `url` = resolved src |

**Image JSON:**
```json
{
  "id": "idx:src/lib/components/Header.svelte:img:0",
  "label": "Company Logo",
  "url": "/images/logo.svg",
  "width": 200,
  "height": 40,
  "opacity": 1.0
}
```

- `width` / `height` — extract from explicit attributes (`width="200"`, `w-[200px]`), CSS, or inline styles. Set to `null` if not determinable.
- `opacity` — extract from CSS `opacity` property or Tailwind `opacity-*` class. Default to `1.0`.
- Images are **visual-only** nodes — they create no edges. They provide context for canvas rendering.
- To import images into FlowSpec, use `flowspec_upload_image` MCP tool (not `flowspec_create_node`).

### Screens

A Screen represents a page or view in the application. Extract from route definitions:

| Source Pattern | Screen |
|---|---|
| SvelteKit routes (`src/routes/**/+page.svelte`) | One screen per route |
| Next.js pages (`app/**/page.tsx`, `pages/**/*.tsx`) | One screen per route |
| Vue routes (`src/views/*.vue` + router config) | One screen per route |
| Generic route configs (React Router, etc.) | One screen per route entry |

**Building regions from source:** Each screen gets regions by analysing the page component's template structure:
1. Identify top-level sections in the page template (header, nav, main content, sidebar, footer, modals).
2. Each distinct section becomes a region. The region's `label` comes from the semantic element, component name, or aria-label.
3. Assign `position` and `size` percentages by estimating from the page layout (use the component's `layout` data). For example, in a column layout: header gets `{x:0, y:0, size:{width:100, height:10}}`, main gets `{x:0, y:10, size:{width:100, height:80}}`, footer gets `{x:0, y:90, size:{width:100, height:10}}`.
4. Populate `elements` by collecting all DataPoints rendered or captured within that section, in template order. Set `order` to preserve sequence.
5. Link `componentNodeId` to the Component that owns the region.

**Screen JSON:**
```json
{
  "id": "idx:screens:login",
  "name": "Login Page",
  "layout": {
    "type": "flex",
    "direction": "column",
    "areas": ["header", "main"]
  },
  "regions": [
    {
      "id": "idx:screens:login:region:0",
      "label": "Login Form",
      "position": { "x": 25, "y": 20 },
      "size": { "width": 50, "height": 60 },
      "elements": [
        { "nodeId": "idx:src/routes/login/+page.svelte:dp:0", "nodeLabel": "User Email", "nodeType": "datapoint", "order": 0 },
        { "nodeId": "idx:src/routes/login/+page.svelte:dp:1", "nodeLabel": "Password", "nodeType": "datapoint", "order": 1 }
      ],
      "componentNodeId": "idx:src/routes/login/+page.svelte:comp:0"
    }
  ]
}
```

- `layout` — page-level layout structure (same schema as Component `layout`), inferred from the route's top-level template
- `regions[].elements[].order` — the order the element appears in the source template (maps to `position_order` in the database). Preserves the visual sequence so the canvas can render fields in the same order as the original UI.
- `regions[].position` — percentage (0-100) relative to the screen, estimating where the region sits. Infer from layout structure: a header region gets `y: 0`, a sidebar gets `x: 0`, a main content area gets centred values.
- `regions[].size` — percentage (0-100) of the screen this region occupies. Infer from layout: a full-width header might be `{ width: 100, height: 10 }`, a sidebar `{ width: 25, height: 90 }`, a card form `{ width: 50, height: 60 }`.

### DataFlow (Edges)

Edges represent how data moves between elements. Extract from:

| Pattern | Edge |
|---|---|
| Component renders a DataPoint (displays) | `from: datapoint, to: component, edgeType: "flows-to"` |
| Component captures a DataPoint (captures) | `from: component, to: datapoint, edgeType: "flows-to"` |
| Transform reads a DataPoint (input) | `from: datapoint, to: transform, edgeType: "transforms"` |
| Transform produces a DataPoint (output) | `from: transform, to: datapoint, edgeType: "flows-to"` |
| DataPoint derived from another | `from: source, to: derived, edgeType: "derives-from"` |
| Validation checks a DataPoint | `from: transform, to: datapoint, edgeType: "validates"` |
| Component reads from Table (API/DB fetch) | `from: table, to: component, edgeType: "flows-to"` |
| Transform writes to Table (API/DB mutation) | `from: transform, to: table, edgeType: "flows-to"` |
| Parent component renders child component (import + template use) | `from: parent, to: child, edgeType: "contains"` |
| Cross-file imports (component imports util, page loads data) | Connect by matching referenced IDs |

**Edge JSON:**
```json
{
  "from": "idx:src/routes/login/+page.svelte:dp:0",
  "to": "idx:src/lib/auth.ts:tx:0",
  "edgeType": "transforms",
  "label": "login credential"
}
```

**Edge type selection:**
- `"flows-to"` — default data movement (A provides data to B)
- `"derives-from"` — B is computed from A (derived/computed values)
- `"transforms"` — data passes through a transform/processing step
- `"validates"` — a validation checks or constrains data
- `"contains"` — containment relationship: screen contains component (auto-generated for screen regions), or parent component contains child component (auto-generated from `children` arrays). Use `from: parent, to: child`.

## Step-by-Step Procedure

### Step 1: Discover or Load Index

```bash
# Check for existing index
cat .flowspec/codebase-index.json 2>/dev/null
```

If the file exists, parse it. If not, initialise a new index structure:

```json
{
  "version": "1.0.0",
  "createdAt": "<now>",
  "lastRunAt": "<now>",
  "projectRoot": "<cwd>",
  "config": { "include": [], "exclude": [] },
  "files": {},
  "spec": null
}
```

### Step 2: Discover Source Files

Use `find` or glob to list all indexable source files. Record each file's `mtime`.

### Step 3: Diff

For each discovered file:
- If not in the index → mark as **new** (needs indexing)
- If in the index but `mtime > lastIndexed` → mark as **modified** (needs re-indexing)
- If in the index and `mtime <= lastIndexed` → **skip**
- If in the index but not on disk → **remove** from index

### Step 4: Analyse Changed Files

For each new/modified file:
1. Read the file contents
2. Identify the file's role (UI component, API route, utility, schema, etc.)
3. Extract elements following the Extraction Rules above
4. Assign deterministic IDs using the `idx:<path>:<type>:<ordinal>` format
5. Store the `elements` object in the file's index entry
6. Set `lastIndexed` to the current timestamp
7. Set `fileModified` to the file's `mtime`

**ID stability and ordinal assignment:**

- The `ordinal` in `idx:<path>:<type>:<ordinal>` is the **order of appearance** in the source file (0-indexed). The first DataPoint found in a file is `:dp:0`, the second is `:dp:1`, etc.
- **Stability guarantee:** If a file is re-indexed and the elements appear in the same order, they receive the same IDs. Edges referencing those IDs remain valid across re-indexes.
- **Limitation:** If elements are reordered, added, or removed, ordinals shift for all subsequent elements of that type. An element that was `:dp:2` may become `:dp:1` after a deletion.
- **Stale edge handling:** During Step 5.8 (edge validation), any edge referencing an ID that no longer exists is removed. Before removal, attempt **label-based recovery**: search for a node with the same `label` in the same file. If found, log the ID change (`"dp:2 → dp:1 (label: User Email)"`) in the report. Do not auto-fix — report only, so the user can review.
- **Guidance:** Prefer `label` + file path for display and human communication. Use IDs only for edge references and internal lookups.

### Step 5: Rebuild Aggregated Spec

Merge all per-file elements into a single FlowSpec v1.2.0 spec:

1. Collect all `dataPoints` from all files into `spec.dataPoints`
2. Collect all `components` from all files into `spec.components`
3. Collect all `transforms` from all files into `spec.transforms`
4. Collect all `tables` from all files into `spec.tables`
5. Collect all `images` from all files into `spec.images`
6. Collect all `dataFlow` edges from all files into `spec.dataFlow`
7. Collect all `screens` from all files into `spec.screens`
8. **Cross-file edge resolution** — 5-phase algorithm:

   **Phase A: Build the import graph.** For every indexed file, parse its import statements. Resolve each import path to a relative file path in the index (handle `$lib/`, `@/`, `~/` aliases, `index.ts` barrel files, and `* as namespace` imports). Build a map: `{ importingFile → [{ importedFile, importedIdentifiers[] }] }`.

   **Phase B: Match imported identifiers to indexed elements.** For each imported identifier (function name, component name, variable name), search the target file's extracted elements for a node whose `label` matches (case-insensitive). Record matches as `{ importingFileElement, importedFileElement, relationship }`.

   **Phase C: Create edges based on element type × usage pattern.**

   | Imported element type | Usage in importing file | Edge |
   |---|---|---|
   | Component | Rendered in template (`<Comp>`) | `from: parent-comp, to: child-comp, edgeType: "contains"` |
   | Transform (function) | Called with DataPoint args | `from: datapoint, to: transform, edgeType: "transforms"` |
   | DataPoint (exported var) | Read in template or logic | `from: datapoint, to: component, edgeType: "flows-to"` |
   | Table (store/API client) | Queried or mutated | `from: table, to: component/transform, edgeType: "flows-to"` |

   **Phase D: Framework-specific patterns.** Check for these cross-file patterns and add appropriate edges:
   - **SvelteKit:** `+page.server.ts` `load()` return → `+page.svelte` `$props().data` — connect load's output DataPoints to page component's displays
   - **React/Next.js:** `getServerSideProps` / `loader` return → page component props — same pattern
   - **Vue:** `provide()` in parent → `inject()` in child — connect the provided DataPoint to the consuming component
   - **Shared stores:** Svelte stores (`writable`), React context, Zustand/Pinia — connect the store (Table node) to all components that subscribe

   **Phase E: Deduplication.** Before adding any cross-file edge, check if an edge with the same `from`, `to`, and `edgeType` already exists. Skip duplicates.

9. **Validate edges:** Remove any edge where `from` or `to` references an ID that no longer exists in the merged spec. Before removal, attempt label-based recovery (see ID stability section above).
10. **Populate `locations`:** For each Component in `spec.components`, iterate its `displays` array — for each DataPoint ID, add `{ "component": "<comp-id>", "role": "output" }` to that DataPoint's `locations`. Then iterate its `captures` array — for each DataPoint ID, add `{ "component": "<comp-id>", "role": "input" }` to that DataPoint's `locations`. Deduplicate entries with the same component + role.
11. Update `spec.metadata` with counts.

### Step 6: Write Index File

```bash
mkdir -p .flowspec
# Write the complete index
```

Write the full index JSON to `.flowspec/codebase-index.json` with 2-space indentation.

### Step 7: Report

Print a summary:

```
FlowSpec Codebase Index — complete

  Files scanned:    15 (8 new, 4 updated, 3 skipped, 2 removed)
  DataPoints:       42
  Components:       12
  Transforms:        8
  Tables:            3
  Images:            7
  Screens:           5
  Edges:            31

  Index: .flowspec/codebase-index.json
  Spec:  .flowspec/codebase-index.json → spec (v1.2.0)

  To import into FlowSpec:
    flowspec_create_project({ name: "<project> (indexed)" })
    # Then use the spec.dataPoints, spec.components, etc. with flowspec_create_node
```

## Incremental Re-runs

On subsequent invocations, the skill:

1. Loads the existing index
2. Only processes files with `mtime > lastIndexed`
3. Preserves elements from unchanged files
4. Rebuilds the aggregated spec from all files (changed + unchanged)
5. Reports only what changed

This makes re-indexing fast — a 500-file project where 3 files changed only reads those 3 files.

## Edge Cases

### Binary / Non-text Files
Skip silently. The file selection patterns already exclude most binary types.

### Very Large Files
Index fully — every variable, every function, every data flow. Large files often contain the most important application logic. If a file is too large to read in one pass, read it in chunks and merge the extracted elements. Never skip or summarise.

### Generated Files
Skip files that contain `@generated`, `// auto-generated`, or similar markers in the first 5 lines.

### Monorepo / Multi-package
If a `workspaces` key is present in `package.json`, index each workspace separately and prefix IDs with the workspace name: `idx:<workspace>:<path>:<type>:<ordinal>`.

### Empty Projects
If no indexable files are found, create the index with an empty `files` object and a null `spec`. Report: "No indexable source files found."

## Using the Index with FlowSpec MCP

### Create a New Project from Index

```
flowspec_create_project({ name: "My App (indexed)" })
# Returns projectId

# Create nodes — flowspec_create_node only accepts: datapoint, component, transform, table
for each dataPoint in spec.dataPoints:
  flowspec_create_node({ projectId, type: "datapoint", label: dataPoint.label, data: { ... } })

for each component in spec.components:
  flowspec_create_node({ projectId, type: "component", label: component.label, data: { ... } })

for each transform in spec.transforms:
  flowspec_create_node({ projectId, type: "transform", label: transform.description, data: { ... } })

for each table in spec.tables:
  flowspec_create_node({ projectId, type: "table", label: table.label, data: { ... } })

# Create screens — use flowspec_create_screen (NOT flowspec_create_node)
for each screen in spec.screens:
  flowspec_create_screen({ projectId, name: screen.name, layout: screen.layout })
  # Returns screenId
  for each region in screen.regions:
    flowspec_add_region({ projectId, screenId, label: region.label, position: region.position, size: region.size, elements: region.elements })

# Upload images — use flowspec_upload_image (NOT flowspec_create_node)
for each image in spec.images:
  flowspec_upload_image({ projectId, label: image.label, url: image.url, width: image.width, height: image.height })

# Create edges
for each edge in spec.dataFlow:
  flowspec_create_edge({ projectId, source: edge.from, target: edge.to, edgeType: edge.edgeType })

# Auto-layout after import
flowspec_auto_layout({ projectId })
```

> **Important:** `flowspec_create_node` only accepts `datapoint`, `component`, `transform`, and `table` as the `type` parameter. Screens and images have their own dedicated MCP tools.

### Append to Existing Project

```
flowspec_search_nodes({ projectId, query: "..." })  # Check what already exists
# Only create nodes/edges that don't already exist
```

### Analyse After Import

```
flowspec_analyse_project({ projectId })
# Reports orphan nodes, missing connections, duplicates
```

## Configuration

Optional `.flowspec/index-config.json` to override defaults:

```json
{
  "include": ["src/**/*.ts", "src/**/*.svelte"],
  "exclude": ["src/**/*.test.ts", "src/**/*.spec.ts", "**/*.d.ts"],
  "maxFileSize": 100000,
  "skipGenerated": true
}
```

If this file exists, use it. Otherwise, use the defaults from File Selection above.
