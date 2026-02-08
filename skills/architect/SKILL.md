---
name: architect
description: >-
  ORCHESTRATED workflow for creating FlowSpec data architecture projects.
  Multi-step state machine with checkpoints, interview loops, and progressive
  refinement. Three use cases: wireframe overlay, codebase mapping, new screen design.
  Requires FlowSpec MCP tools (flowspec_*).
argument_hint: "[wireframe images | codebase path | 'new project']"
---

# FlowSpec Architect (Orchestrated)

Creates FlowSpec projects — the reverse direction of `spec`/`wireframe`. This skill produces FlowSpec projects from wireframes, codebases, or design sessions using a **state-based orchestration workflow** with quality gates and interview loops.

**Core pattern:** INIT → ROUTE → GATHER → INTERVIEW → GENERATE → VALIDATE → REFINE → FINALIZE

**Requires:** `flowspec_*` MCP tools (FlowSpec desktop app or standalone MCP server).

**Key Features:**
- Screen-by-screen interview loops with user confirmation
- Progressive YAML assembly (mini-YAMLs → merged spec)
- Quality gates with enhanced analysis (naming, fuzzy duplicates, subgraphs)
- Semantic positioning with natural language commands
- Resume capability for interrupted workflows

---

## State Machine Overview

```
INIT (verify MCP, select project)
  ↓
ROUTE (detect use case: UC1/UC2/UC3)
  ↓
GATHER (collect inputs: images/codebase/designs)
  ↓
INTERVIEW (iterative user clarification, screen-by-screen)
  ↓
GENERATE (build YAML, progressive assembly)
  ↓
VALIDATE (enhanced analysis, quality checks)
  ↓
REFINE (layout, regions, smart positioning)
  ↓
FINALIZE (export YAML, generate tech spec)
```

**State Persistence:** Use Task system (TaskCreate/TaskUpdate) to track state and enable resume.

---

## Phase 1: INIT (Setup & Verification)

### Checkpoint 1.1: Verify MCP Connection
```
1. Call flowspec_list_projects()
2. If error → tell user to open FlowSpec desktop app or start MCP server
3. Display available projects (name, updated_at)
```

### Checkpoint 1.2: Project Selection
Ask user:
- **Create new project?** → `flowspec_create_project(name)`
- **Use existing project?** → user selects from list
- **Clone for safety?** → `flowspec_clone_project(projectId)` (recommended)

Save `projectId` to state.

### Checkpoint 1.3: State Transition
```
✓ Project ready → state: ROUTE
```

---

## Phase 2: ROUTE (Determine Use Case)

Analyse user input to determine workflow:

| User provides | Route |
|---------------|-------|
| Wireframe images (PNG/JPG) | **UC1: Wireframe → Dataflow** |
| Codebase path (directory) | **UC2: Codebase → Dataflow** |
| "Design screens" / "new project" | **UC3: Design New Screens** |
| YAML file | Direct import (skip to VALIDATE) |

Save `useCase` to state.

### Checkpoint 2.1: State Transition
```
✓ Use case determined → state: GATHER
```

---

## UC1: Wireframe → Dataflow (Interview-Driven)

**Goal:** Overlay wireframes with data architecture through screen-by-screen interviews.

### Phase 3: GATHER (Upload & Create Screens)

For each wireframe image:

**Step 3.1:** Upload image
```
result = flowspec_upload_image(filePath: "/path/to/wireframe.png")
→ { url, width, height, filename }
```

**Step 3.2:** Create screen
```
screen = flowspec_create_screen(
  projectId,
  name: "Login Page",
  imageUrl: result.url,
  imageWidth: result.width,
  imageHeight: result.height,
  imageFilename: result.filename
)
→ { id, name }
```

Save `screenId` for later region linking.

**Checkpoint 3.1:** All screens created
```
✓ Screens: ${screenIds.length} created → state: INTERVIEW
```

---

### Phase 4: INTERVIEW (Screen-by-Screen)

**Critical:** Interview each screen individually with user confirmation loops.

For each screen:

**Step 4.1:** Show wireframe via vision
- Display the wireframe image (use Read tool if needed)
- Identify distinct UI regions:
  - Headers, sidebars, navigation
  - Forms (input fields, buttons)
  - Data displays (tables, cards, charts)
  - Interactive elements (dropdowns, toggles)

**Step 4.2:** Region-by-region interview
For each identified region, ask user:

```
Q1: "What data elements are in this region?"
   → List field names, labels, values

Q2: "What are the data types?"
   → string, number, boolean, object, array

Q3: "Is this data captured (user input) or inferred (computed/fetched)?"
   → captured: form inputs, selections
   → inferred: calculations, API responses, database queries

Q4: "What constraints apply?"
   → required, enum, range, format, uniqueness

Q5: "What business logic affects this data?"
   → validations, calculations, workflows
```

**Step 4.3:** Build mini-YAML for this screen
Assemble a mini-YAML fragment:

```yaml
# Mini-YAML for "Login Page"
dataPoints:
  - id: email_input
    label: Email Address
    type: string
    source: captured
    constraints: [required, email-format]

  - id: auth_token
    label: Auth Token
    type: string
    source: inferred
    sourceDefinition: "API /auth/login response"

components:
  - id: login_page
    label: Login Page
    displays: [auth_token]
    captures: [email_input, password_input]

transforms:
  - id: validate_email
    label: Validate Email Format
    type: validation
    inputs: [email_input]
    outputs: [email_valid]
    logic:
      type: formula
      content: "regex match @example.com"
```

**Step 4.4:** User confirmation checkpoint
```
ASK: "Screen '${screenName}' complete? [yes / no / refine]"

- yes → save mini-YAML, move to next screen
- no → re-interview this screen (go back to Step 4.2)
- refine → ask specific clarifications, update mini-YAML
```

**Checkpoint 4.1:** All screens interviewed
```
✓ Mini-YAMLs collected: ${miniYamls.length} → state: GENERATE
```

---

### Phase 5: GENERATE (YAML Assembly)

**Step 5.1:** Merge mini-YAMLs
- Combine all screen mini-YAMLs into full spec
- Deduplicate node IDs (same label + type → single node)
- Merge constraints and references

**Step 5.2:** Add cross-screen dataFlow edges
Trace data paths between screens:
- Component A captures → Component B displays (flows-to)
- Transform produces → Component displays (derives-from)
- Validation checks → DataPoint (validates)

**Step 5.3:** Add tables (if applicable)
If database tables or API endpoints were mentioned:
```yaml
tables:
  - id: users_table
    label: Users
    columns:
      - name: email
        type: string
      - name: created_at
        type: string
    sourceType: database
    endpoint: "postgresql://users"
```

**Step 5.4:** Add screens metadata
Include screen IDs from Phase 3 (regions added in REFINE):
```yaml
screens:
  - id: screen_1
    name: "Login Page"
    imageUrl: "https://..."
    imageWidth: 1920
    imageHeight: 1080
    regions: []  # Added in REFINE phase
```

**Checkpoint 5.1:** YAML generated
```
✓ Full YAML spec assembled → state: VALIDATE
```

---

### Phase 6: VALIDATE (Quality Checks)

**Step 6.1:** Import YAML with auto-layout
```
flowspec_import_yaml(
  projectId,
  yaml: fullYaml,
  autoLayout: true,
  layoutDirection: "TB"
)
```

**Step 6.2:** Run enhanced analysis
```
analysis = flowspec_analyse_project(projectId)
```

Check for:
- ✅ Orphan nodes (no edges)
- ✅ Exact duplicate labels
- ✅ **Naming convention violations** (mixed-case, special chars, inconsistent prefixes)
- ✅ **Fuzzy duplicates** (near matches via Levenshtein/cosine)
- ✅ **Unreachable subgraphs** (isolated clusters)
- ✅ **Consolidation opportunities** (same type + constraints)

**Step 6.3:** Quality gate decision
```
IF issues found:
  → Display analysis results
  → Ask user: "Fix automatically [auto] or manually [manual] or skip [skip]?"

  - auto → apply suggested fixes, re-import YAML, re-validate
  - manual → prompt specific fixes, user edits, re-import
  - skip → proceed with warnings

  → state: REFINE (on success) or VALIDATE (retry)

ELSE:
  → state: REFINE
```

**Checkpoint 6.1:** Validation passed
```
✓ Quality checks passed → state: REFINE
```

---

### Phase 7: REFINE (Regions & Layout)

**Step 7.1:** Add regions to screens
For each screen and identified UI region from INTERVIEW phase:

```
flowspec_add_region(
  projectId,
  screenId,
  label: "Login Form",
  position: { x: 10, y: 20 },  # percentage (0-100)
  size: { width: 80, height: 60 },
  elementIds: [nodeId1, nodeId2]  # node IDs from canvas
)
```

**Position guidelines:**
- Use percentage coordinates (0-100) relative to wireframe
- Top-left = (0, 0), bottom-right = (100, 100)
- Typical sizes: header (100, 10), sidebar (20, 80), form (60, 40)

**Step 7.2:** Semantic layout (optional)
Offer smart positioning commands:

```
ASK: "Apply semantic layout? Examples:
- 'move auth nodes to top-right'
- 'arrange payment flow vertically on left'
- 'cluster validation transforms in bottom-left'
"

IF user provides command:
  flowspec_smart_layout(projectId, command)
```

**Step 7.3:** Manual layout adjustment (optional)
```
ASK: "Re-run auto-layout with different direction? [TB/BT/LR/RL/skip]"

IF user selects direction:
  flowspec_auto_layout(projectId, direction)
```

**Checkpoint 7.1:** Refinement complete
```
✓ Regions linked, layout optimized → state: FINALIZE
```

---

### Phase 8: FINALIZE (Export & Tech Spec)

**Step 8.1:** Export final YAML
```
yaml = flowspec_get_yaml(projectId)
```

Save to file: `{projectName}_flowspec.yaml`

**Step 8.2:** Generate tech spec markdown
Create structured implementation guide:

```markdown
# {Project Name} - Technical Specification

Generated: {timestamp}

## DataPoints (${dataPoints.length})

| Label | Type | Source | Constraints |
|-------|------|--------|-------------|
| Email Address | string | captured | required, email-format |
| Auth Token | string | inferred | - |

## Components (${components.length})

| Label | Displays | Captures | Wireframe |
|-------|----------|----------|-----------|
| Login Page | auth_token | email_input, password_input | Login Page |

## Transforms (${transforms.length})

| Label | Type | Inputs | Outputs | Logic |
|-------|------|--------|---------|-------|
| Validate Email | validation | email_input | email_valid | regex match |

## Tables (${tables.length})

| Label | Source | Columns |
|-------|--------|---------|
| Users | postgresql | email, created_at |

## Implementation Recommendations

1. **Database Schema**: Create tables with listed columns and types
2. **API Contracts**: Build endpoints for inferred DataPoints (auth_token, etc.)
3. **Component Structure**: Implement UI components with listed displays/captures
4. **Validation Rules**: Apply constraints at both client and server
5. **Data Flow**: Follow edges for state management and data passing

## Quality Metrics

- Total nodes: ${totalNodes}
- Connected nodes: ${connectedNodes}
- Orphans: ${orphans}
- Duplicate labels: ${duplicates}
- Naming violations: ${namingViolations}
- Fuzzy duplicates: ${fuzzyDuplicates}

✅ Ready for implementation in Claude Code via `/spec` skill
```

**Step 8.3:** Completion summary
```
✅ FlowSpec project created successfully!

- Project: ${projectName}
- Screens: ${screenCount}
- Nodes: ${nodeCount} (DataPoints: ${dp}, Components: ${comp}, Transforms: ${tf})
- Edges: ${edgeCount}
- YAML: ${yamlFilePath}
- Tech Spec: ${specFilePath}

Next steps:
1. Review YAML in FlowSpec desktop app
2. Use tech spec for implementation planning
3. Generate code with: /spec ${yamlFilePath}
```

**Checkpoint 8.1:** Workflow complete
```
✓ FINALIZE done → state: DONE
```

---

## UC2: Codebase → Dataflow (AST-Driven)

**Goal:** Reverse-engineer existing codebase into FlowSpec architecture.

See [codebase-analysis.md](references/codebase-analysis.md) and [ast-parsing.md](references/ast-parsing.md) for detailed AST parsing strategies.

### Phase 3: GATHER (Codebase Analysis)

**Step 3.1:** Scope assessment
```
1. Detect framework (check package.json, directory structure)
   - SvelteKit: src/routes/, +page.svelte
   - Next.js: pages/ or app/, React
   - Express: server.js, routes/
   - Remix: app/routes/

2. Count routes/pages (determine granularity)
   - < 10 pages → single YAML
   - 10-30 pages → split by domain (auth, admin, public)
   - > 30 pages → per-module YAMLs

3. Identify database/API access
   - SQL queries, ORM models
   - API fetch calls, endpoints
```

**Step 3.2:** Check existing specs
```
projects = flowspec_list_projects()
existingNodes = flowspec_search_nodes(query: "codebase-relevant-term")

IF existing project found:
  ASK: "Update existing project or create new?"
```

**Checkpoint 3.1:** Scope defined
```
✓ Framework: ${framework}, Routes: ${routeCount} → state: INTERVIEW
```

---

### Phase 4: INTERVIEW (AST Guidance)

**Step 4.1:** AST parsing strategy
Explain to user what will be analysed:

```
"I'll parse your ${framework} codebase using AST (Abstract Syntax Tree) analysis:

1. **Type Definitions** → DataPoints
   - Interfaces, types, Zod schemas
   - Extract properties, types, constraints

2. **Routes/Pages** → Components
   - Each route file → one Component
   - Identify load functions (data fetching)
   - Identify form actions (data capture)

3. **Utilities** → Transforms
   - Functions in lib/utils, lib/services
   - Calculate, validate, workflow logic

4. **Data Flow** → Edges
   - Trace imports, function calls
   - Follow data dependencies

Is this approach okay? [yes / customize]"
```

**Step 4.2:** AST parsing execution
Use AST parsing techniques from [ast-parsing.md](references/ast-parsing.md):

- **TypeScript:** `typescript` compiler API
- **Svelte:** `svelte/compiler` + regex for runes
- **React:** `@babel/parser` + `@babel/traverse`

**Step 4.3:** Progressive mini-YAML generation
For each module/route:
1. Parse file → extract entities
2. Build mini-YAML fragment
3. **User confirmation:** "Module '${moduleName}' mapped. Correct? [yes/no/refine]"
4. Save mini-YAML

**Checkpoint 4.1:** All modules parsed
```
✓ Mini-YAMLs: ${miniYamls.length} → state: GENERATE
```

---

### Phase 5-8: Same as UC1
Follow GENERATE → VALIDATE → REFINE → FINALIZE phases.

**Additional Step in REFINE (UC2-specific):**
```
ASK: "Capture screenshots of running app for screen overlay? [yes/no]"

IF yes:
  - User provides localhost URL or deployed URL
  - Use Playwright capture (if P1 tool available):
    flowspec_capture_screen(url, selector, viewport)
  - Or user uploads screenshots manually:
    flowspec_upload_image(filePath)

  - Create screens + add regions (same as UC1 Phase 7)
```

---

## UC3: Design New Screens (Collaborative)

**Goal:** Design new screens from scratch or extend existing project.

### Phase 3: GATHER (Design Input)

**Step 3.1:** Obtain screen design
```
ASK: "How would you like to design the screen?
1. Upload image from Figma/Sketch/Pencil
2. Start with blank screen (text description)
3. Use existing screen as template
"
```

**Step 3.2:** Upload or create
- **If image:** Same as UC1 Phase 3 (upload → create screen)
- **If blank:** Skip image, create screen with placeholder
- **If template:** Clone existing screen

**Checkpoint 3.1:** Screen created
```
✓ Screen: ${screenName} → state: INTERVIEW
```

---

### Phase 4: INTERVIEW (Collaborative Design)

**Step 4.1:** Region definition
Walk through screen with user:

```
"Let's define UI regions for '${screenName}':

For each region (header, form, table, sidebar, etc.):
1. Region name: _____
2. Position (x%, y%): _____, _____
3. Size (width%, height%): _____, _____
4. Data elements: [list]
5. Interactions: [buttons, links, inputs]
"
```

**Step 4.2:** Data element specification
For each data element in region:
```
Q1: Label? → _____
Q2: Type? → string/number/boolean/object/array
Q3: Source? → captured/inferred
Q4: Constraints? → required, enum, range, format
Q5: Related transforms? → validation, calculation, workflow
```

**Step 4.3:** Build mini-YAML
Same as UC1 Phase 4 Step 4.3.

**Step 4.4:** Confirmation
```
ASK: "Screen design complete? [yes/no/add-more-regions]"
```

**Checkpoint 4.1:** Design complete
```
✓ Regions defined → state: GENERATE
```

---

### Phase 5-8: Same as UC1
Follow GENERATE → VALIDATE → REFINE → FINALIZE phases.

**Merge mode in GENERATE (UC3-specific):**
```
IF extending existing project:
  flowspec_import_yaml(projectId, yaml: miniYaml, merge: true)
  # Merges new nodes with existing, preserves structure
```

---

## Quality Gate Checklist

Run at **every VALIDATE checkpoint**:

```yaml
VALIDATE_CHECKLIST:
  - orphans.length === 0
    → Every node must have at least one edge

  - duplicates.length === 0
    → No exact duplicate labels

  - namingViolations.length === 0
    → Consistent naming convention (snake_case or camelCase, not mixed)

  - fuzzyDuplicates.length === 0
    → No near-duplicates (orderTotal vs order_total)

  - isolatedSubgraphs.length === 0
    → All nodes connected to main graph

  - capturedDataPoints.every(dp =>
      components.some(c => c.captures.includes(dp.id))
    )
    → Every captured DataPoint has capturing Component

  - inferredDataPoints.every(dp =>
      transforms.some(t => t.outputs.includes(dp.id)) ||
      components.some(c => c.displays.includes(dp.id))
    )
    → Every inferred DataPoint has producing Transform or fetching Component

IF any check fails:
  → state: REFINE
  → Display specific failures
  → Suggest fixes
  → Re-validate after fixes
```

---

## State Machine Error Recovery

**Interrupted workflow?** Resume from last checkpoint:

```
1. Check Task list for latest state
2. Retrieve saved context (projectId, useCase, miniYamls, screenIds)
3. Resume from last completed phase:
   - INIT → start over (quick)
   - ROUTE → start over (quick)
   - GATHER → resume uploads/parsing
   - INTERVIEW → resume from last confirmed screen
   - GENERATE → regenerate from saved mini-YAMLs
   - VALIDATE → re-run analysis
   - REFINE → continue layout/regions
   - FINALIZE → re-export
```

**State transition failure?**
```
IF state transition blocked:
  1. Display error details
  2. Suggest corrective action
  3. Allow manual state override (with confirmation)
  4. Offer rollback to previous checkpoint
```

---

## Advanced Features

### Semantic Positioning Examples
```
# Move specific node types to regions
"move all validation transforms to bottom-left"
"arrange payment flow vertically on right side"
"cluster auth-related nodes in top-right"

# Organize by relationship
"group captured DataPoints near their Components"
"stack transforms between their inputs and outputs"
```

### Progressive Refinement
```
1. Import initial YAML (coarse-grained)
2. Run analysis → identify gaps
3. Add missing nodes/edges incrementally
4. Re-import with merge: true
5. Validate again → iterate
```

### Multi-YAML Projects
```
# For large codebases
1. Generate per-domain YAMLs (auth.yaml, admin.yaml, public.yaml)
2. Import each with merge: true
3. Deduplicate shared DataPoints
4. Consolidate cross-domain edges
```

---

## Tool Reference

See [yaml-schema.md](references/yaml-schema.md) for v1.2.0 YAML structure.

See [workflow-orchestration.md](references/workflow-orchestration.md) for state machine details.

See [ast-parsing.md](references/ast-parsing.md) for framework-specific parsing strategies.

See [codebase-analysis.md](references/codebase-analysis.md) for UC2 full pipeline.

---

## Success Criteria

A FlowSpec project is **complete** when:

✅ All phases reached FINALIZE state
✅ Quality gate passed (0 issues)
✅ YAML exported to file
✅ Tech spec generated
✅ Regions linked to screens (if wireframes used)
✅ Layout optimized (auto-layout or smart layout applied)
✅ User confirmed satisfaction

**Output artifacts:**
- `${projectName}_flowspec.yaml` (importable spec)
- `${projectName}_tech_spec.md` (implementation guide)
- FlowSpec project (viewable in desktop app)

**Next step:** Generate implementation code with `/spec ${projectName}_flowspec.yaml`
