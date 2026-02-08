# Codebase Analysis Pipeline

An 8-step methodology for mapping an existing codebase's data architecture into a FlowSpec project.

---

## Step 1: Scope Assessment

### Actions
1. Count **routes/pages** (the UI surface area)
2. Count **utility/service modules** (the logic layer)
3. Count **database tables or API endpoints** (the data layer)
4. Identify **domains** — logical groupings of related functionality

### Granularity Decision Tree

| Routes | Tables/Endpoints | Strategy |
|--------|-----------------|----------|
| < 10   | < 10            | Single YAML, no domain grouping |
| 10-30  | 10-30           | Single YAML with domain sections (comments) |
| 30+    | 30+             | Split into separate YAML per domain |

### Framework Detection

Identify the framework to know where to look for data:

| Framework | Routes | Server Logic | Types |
|-----------|--------|-------------|-------|
| **SvelteKit** | `src/routes/**/+page.svelte` | `+page.server.ts`, `+server.ts` | `src/lib/types.ts` |
| **Next.js** (App Router) | `app/**/page.tsx` | `app/**/route.ts`, API routes | Types near components |
| **Next.js** (Pages) | `pages/**/*.tsx` | `pages/api/**` | `types/` or `lib/types.ts` |
| **Express/Fastify** | N/A (API only) | Route handlers | `types/` or inline |
| **Remix** | `app/routes/**` | Loaders/actions in same file | `app/types.ts` |
| **Nuxt** | `pages/**/*.vue` | `server/api/**` | `types/` |

### Framework-Specific Guidance

**SvelteKit:**
- Types: `src/lib/types.ts` or `$lib/types`
- Server data: `+page.server.ts` (load function), `+server.ts` (API)
- Forms: SvelteKit form actions in `+page.server.ts`, `<form>` elements in `+page.svelte`
- State: Look for `$state`, `$derived`, stores
- Utils: `src/lib/utils/` or `$lib/`

**Next.js (App Router):**
- Types: `types/` directory or co-located with components
- Server data: Server Components (default), Route Handlers in `route.ts`
- Forms: Server Actions, `useFormState`, client-side forms
- State: React state, context, Zustand/Redux stores
- Utils: `lib/` or `utils/` directories

**Next.js (Pages Router):**
- Types: `types/` directory
- Server data: `getServerSideProps`, `getStaticProps`, API routes in `pages/api/`
- Forms: Client-side with `fetch` to API routes
- Utils: `lib/` or `utils/`

**Express / Fastify (API-only):**
- Types: `types/` or inline in route handlers
- No UI components — document API endpoints as "external" Components
- Transforms are the route handler logic
- Focus on request/response data shapes

**Remix:**
- Types: Co-located or in `app/types.ts`
- Server data: `loader` functions (GET), `action` functions (POST)
- Forms: Remix `<Form>` component with actions
- Everything co-located in route files

---

## Step 2: Check Existing Specs

Before analysing the codebase, check if a FlowSpec project already exists:

1. Call `flowspec_list_projects` to see all existing projects
2. If a project exists for this codebase, call `flowspec_get_yaml` with its `projectId` to retrieve it
3. Use the existing spec as a **baseline** — update rather than recreate from scratch
4. Call `flowspec_search_nodes` with key terms to find matches in other projects

> **Why check first?** Rebuilding a 100+ node spec from scratch wastes effort and loses manual refinements (layout, annotations). Updating an existing spec preserves that work.

---

## Step 3: Discover DataPoints

DataPoints are the **atomic data elements** that flow through the system. Every meaningful field that is either entered by a user or computed by the system is a DataPoint.

### Discovery Order (most reliable -> least)
1. **Type definitions** (`types.ts`, interfaces, schemas) -> Entity fields become DataPoints
2. **Server loaders/handlers** -> SQL queries and API responses reveal which fields actually flow
3. **Form components** -> `<input>`, `<select>`, form actions -> **captured** DataPoints
4. **Calculation outputs** -> Return types of utility functions -> **inferred** DataPoints
5. **Constants** -> Fixed values used in calculations -> **inferred** DataPoints (immutable)

### Classification Rules

| Source | Definition | Visual |
|--------|-----------|--------|
| `captured` | User enters via form, file upload, or direct input | Solid border |
| `inferred` | Computed by code, fetched from API, derived from other data | Dashed border |

**Rule of thumb**: If removing the UI form would make this data disappear, it's `captured`. If the data would still exist (because it's calculated), it's `inferred`.

### Data Type Assignment
- `string` — text, dates (ISO strings), enum values, identifiers
- `number` — currencies, percentages, counts, quantities
- `boolean` — flags, toggles, yes/no choices
- `object` — structured data with named fields (addresses, configuration)
- `array` — lists of items (line items, history entries)

### Constraints
Document meaningful constraints:
- `required` — must be provided
- `enum` — limited set of valid values
- `min X` / `max X` — numeric bounds
- `range X-Y` — bounded range
- `unique` — must be unique within scope
- `date format` — must be a valid date
- `email format`, `url format` — format validations
- `immutable` — cannot change after creation

### Grouping Strategy
When the codebase has multiple domains, **prefix DataPoint IDs** with a domain indicator:
```
dp-company-name      (Companies domain)
dp-investor-type     (Investors domain)
dp-portfolio-moic    (Portfolio domain)
```

---

## Step 4: Discover Components

Components represent **UI pages or significant UI sections** that display or capture data.

### Actions
1. One Component per **page route** (the primary grain)
2. Optionally add Components for **significant sub-sections** (modals, wizards, tabbed panels) if they have distinct data relationships
3. For each Component, determine:
   - **displays**: Which DataPoint IDs does it render/show to the user?
   - **captures**: Which DataPoint IDs does it accept as input from the user?

### Component ID Convention
```
comp-{page-name}          (e.g., comp-dashboard, comp-cap-table)
comp-{page-name}-{section} (e.g., comp-cap-table-import-modal)
```

### Notes
- A Component that only displays data has an empty `captures` array
- A Component that only captures data (pure form) has an empty `displays` array
- Most pages both display AND capture data
- API-only codebases (Express, Fastify) may have zero Components — the "component" equivalent is the API consumer (documented as external)

---

## Step 5: Discover Transforms

Transforms represent **business logic** — functions that take data in and produce data out.

### Actions
1. Scan utility/service modules for **exported functions with business logic**
2. For each Transform, document:
   - **type**: `formula` (calculation), `validation` (checking), or `workflow` (multi-step process)
   - **inputs**: Which DataPoint IDs flow in
   - **outputs**: Which DataPoint IDs flow out
   - **logic**: Algorithm description

### Logic Documentation
```yaml
# Simple formula
logic:
  type: formula
  content: "total = price * quantity * (1 + tax_rate)"

# Decision table (branching logic)
logic:
  type: decision_table
  content:
    condition_a: "result when A"
    condition_b: "result when B"

# Multi-step workflow
logic:
  type: steps
  content: "1) Validate input; 2) Query database; 3) Transform result; 4) Cache and return"
```

### What Counts as a Transform
- **YES**: Functions with business meaning (calculate tax, validate email, distribute proceeds)
- **YES**: Database queries that join/aggregate (they transform raw data into a shape)
- **YES**: External API integrations that fetch and reshape data
- **NO**: Pure formatting functions (display concerns, not data architecture)
- **NO**: Generic CRUD operations with no business logic
- **NO**: Framework boilerplate (routing, middleware)

### Transform ID Convention
```
tx-{descriptive-name}     (e.g., tx-waterfall-distribution, tx-tax-treatment)
```

---

## Step 6: Map DataFlow Edges

Edges connect nodes to show how data flows through the system.

### Edge Types
| Type | Meaning | When to Use |
|------|---------|-------------|
| `flows-to` | Data moves from source to target | Default for most connections |
| `derives-from` | Target is computed from source | Transform -> inferred DataPoint |
| `transforms` | Data passes through a transform | DataPoint -> Transform |
| `validates` | Transform validates a DataPoint | Validation transform -> validated DataPoint |
| `contains` | Table contains a DataPoint | Table -> DataPoint |

### Standard Flow Patterns

**Pattern 1: User input -> Storage**
```
Component (captures) -[flows-to]-> captured DataPoint
```

**Pattern 2: Data -> Calculation -> Result**
```
captured DataPoint -[transforms]-> Transform -[derives-from]-> inferred DataPoint
```

**Pattern 3: Result -> Display**
```
inferred DataPoint -[flows-to]-> Component (displays)
```

**Pattern 4: Validation**
```
Transform -[validates]-> DataPoint (being validated)
```

**Pattern 5: External data fetch**
```
captured DataPoint (ID/key) -[transforms]-> workflow Transform -[derives-from]-> inferred DataPoint (fetched data)
```

**Pattern 6: Table persistence**
```
Table -[contains]-> DataPoint
```

### Tracing Strategy
For each page route, trace the full data path:
1. **What does the server loader query?** -> Identifies DataPoint -> Component edges
2. **What does the form submit?** -> Identifies Component -> DataPoint edges
3. **What utility functions does the page call?** -> Identifies Transform edges
4. **What does the page render?** -> Identifies DataPoint -> Component display edges

---

## Step 7: Produce YAML

Assemble all discovered nodes and edges into FlowSpec YAML v1.2.0 format. See [yaml-schema.md](yaml-schema.md) for the complete schema.

### File Naming
```
flowspec-{project-name}.yaml           (single file)
flowspec-{project-name}-{domain}.yaml  (per-domain split)
```

### Quality Checklist
Before finalising the YAML:
- [ ] Every `captured` DataPoint is captured by at least one Component
- [ ] Every `inferred` DataPoint has at least one Transform producing it
- [ ] Every Transform has at least one input and one output
- [ ] Every Component either displays or captures at least one DataPoint
- [ ] No orphan nodes (every node has at least one edge)
- [ ] IDs are consistent across dataPoints, components, transforms, and dataFlow sections
- [ ] Edge types follow the rules (derives-from for Transform->inferred, transforms for DataPoint->Transform, etc.)

---

## Step 8: Cross-Project Validation

Use MCP tools to check naming consistency and find shared data patterns across all FlowSpec projects.

### Actions
1. Call `flowspec_search_nodes` with key DataPoint labels (e.g., `"email"`, `"company"`, `"user"`) to find matches in other projects
2. Compare naming conventions — flag inconsistencies:
   - Same concept, different name (e.g., `dp-company-name` vs `dp-firm-name`)
   - Same name, different type (e.g., `dp-amount` as `number` in one project, `string` in another)
3. Identify **shared entities** that appear across multiple projects — candidates for a shared data dictionary

---

## Edge Cases

### Shared Components
If a component is reused across multiple pages (e.g., a data table), document it once and reference it from multiple Component nodes via edges.

### Real-time Data
WebSocket or SSE connections -> treat the subscription setup as a `workflow` Transform, and the received data as `inferred` DataPoints.

### Third-party Integrations
External API calls -> `workflow` Transform with the API as an implied external system. Document what goes in (request) and what comes out (response).

### Authentication / Authorisation
Auth data (user session, roles, permissions) -> `captured` DataPoints (user provides credentials) flowing through `validation` Transforms (auth checks).

### File Uploads
Uploaded files -> `captured` DataPoint of type `object` or `string` (URL after upload), with a `workflow` Transform for the upload/processing pipeline.
