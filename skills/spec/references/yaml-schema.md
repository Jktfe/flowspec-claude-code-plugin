# FlowSpec Main YAML Schema

## Version

Current schema version: `1.0.0`

## Top-Level Structure

```yaml
version: "1.0.0"
metadata:
  projectName: string        # Name of the project
  exportedAt: string         # ISO 8601 timestamp
  nodeCount: number          # Total DataPoints + Components + Transforms
  edgeCount: number          # Total edges in dataFlow

dataPoints: DataPoint[]      # All data elements
components: Component[]      # All UI components/pages
transforms: Transform[]      # All business logic nodes
dataFlow: Edge[]             # All connections between nodes
```

---

## DataPoint

An atomic data element that flows through the system.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `string` | Yes | Unique ID. Convention: `dp-{name}` |
| `label` | `string` | Yes | Human-readable name |
| `type` | `DataType` | Yes | One of: `string`, `number`, `boolean`, `object`, `array` |
| `source` | `SourceType` | Yes | One of: `captured`, `inferred` |
| `sourceDefinition` | `string` | Yes | How this data enters the system |
| `constraints` | `string[]` | Yes | Validation rules and bounds (can be empty) |
| `locations` | `Location[]` | Yes | Which Components use this DataPoint |

### Location

| Field | Type | Description |
|-------|------|-------------|
| `component` | `string` | Component ID |
| `role` | `"input" \| "output"` | `input` = Component captures it, `output` = Component displays it |

### DataType Values

| Type | Use For |
|------|---------|
| `string` | Text, dates (ISO strings), enum values, identifiers, URLs |
| `number` | Currency amounts, percentages, counts, quantities, scores |
| `boolean` | Flags, toggles, yes/no choices |
| `object` | Structured data with named fields (address, config) |
| `array` | Lists (line items, history entries, collections) |

### SourceType Values

| Source | Meaning | Implication |
|--------|---------|-------------|
| `captured` | User enters via form, upload, or direct input | Needs a form/input in the UI; stored in database |
| `inferred` | Computed, fetched, or derived from other data | Needs a function/query to produce it; may not be stored |

**Rule of thumb**: If removing the UI form would make this data disappear, it's `captured`. If the data would still exist (because it's calculated), it's `inferred`.

### Common Constraints

- `required` — Must be provided
- `enum` — Limited set of valid values
- `min X` / `max X` — Numeric bounds
- `range X-Y` — Bounded range
- `unique` — Must be unique within scope
- `date format` — Must be a valid date
- `email format`, `url format` — Format validations
- `immutable` — Cannot change after creation

---

## Component

A UI page or significant UI section.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `string` | Yes | Unique ID. Convention: `comp-{name}` |
| `label` | `string` | Yes | Human-readable name |
| `wireframeRef` | `string` | No | Reference to a screenshot/wireframe image |
| `displays` | `string[]` | Yes | DataPoint IDs this component renders (can be empty) |
| `captures` | `string[]` | Yes | DataPoint IDs this component accepts as input (can be empty) |

### Notes

- `displays` and `captures` reference DataPoint IDs
- A display-only page has `captures: []`
- A form-only page has `displays: []`
- Most pages both display AND capture data
- `wireframeRef` is optional — present when wireframe images are attached in FlowSpec

---

## Transform

Business logic that processes data.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `string` | Yes | Unique ID. Convention: `tx-{name}` |
| `type` | `LogicType` | Yes | One of: `formula`, `validation`, `workflow` |
| `description` | `string` | Yes | What this transform does |
| `inputs` | `string[]` | Yes | DataPoint IDs consumed |
| `outputs` | `string[]` | Yes | DataPoint IDs produced |
| `logic` | `Logic` | Yes | Algorithm description |

### Logic

| Field | Type | Description |
|-------|------|-------------|
| `type` | `LogicFormat` | One of: `formula`, `decision_table`, `steps` |
| `content` | `string \| object` | Algorithm description (format depends on `type`) |

### LogicType Values

| Type | Use For |
|------|---------|
| `formula` | Mathematical calculations, aggregations, derivations |
| `validation` | Data validation, constraint checking, business rule enforcement |
| `workflow` | Multi-step processes, API integrations, data pipelines |

### LogicFormat Values

| Format | Content Type | Example |
|--------|-------------|---------|
| `formula` | `string` | `"total = price * qty * (1 + tax_rate)"` |
| `decision_table` | `object` | `{ "EIS": "30% relief", "SEIS": "50% relief" }` |
| `steps` | `string` | `"1) Validate; 2) Query; 3) Transform; 4) Return"` |

---

## DataFlow Edge

A connection between two nodes.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | `string` | Yes | Source node ID (any node type) |
| `to` | `string` | Yes | Target node ID (any node type) |
| `edgeType` | `EdgeType` | Yes | One of: `flows-to`, `derives-from`, `transforms`, `validates` |
| `label` | `string` | No | Describes the connection |

### EdgeType Values

| Type | Meaning | Typical Connection |
|------|---------|-------------------|
| `flows-to` | Data moves from source to target | DataPoint -> Component, Component -> DataPoint |
| `derives-from` | Target is computed from source | Transform -> inferred DataPoint |
| `transforms` | Data passes through a transform | DataPoint -> Transform |
| `validates` | Transform validates target data | validation Transform -> DataPoint |

### Connection Rules

| Source -> Target | Allowed | Notes |
|-----------------|---------|-------|
| DataPoint -> DataPoint | Yes | Must have compatible types |
| DataPoint -> Component | Yes | Data flows to UI |
| DataPoint -> Transform | Yes | Data inputs to logic |
| Transform -> DataPoint | Yes | Computed outputs |
| Transform -> Component | Yes | Direct to display |
| Component -> DataPoint | Yes | Only if target is `captured` |
| Self-connection | No | |
| Duplicate connection | No | Same source + target |

---

## Minimal Example

```yaml
version: "1.0.0"
metadata:
  projectName: Task Manager
  exportedAt: "2026-01-15T10:00:00.000Z"
  nodeCount: 5
  edgeCount: 4

dataPoints:
  - id: dp-task-title
    label: Task Title
    type: string
    source: captured
    sourceDefinition: User enters task title in creation form
    constraints:
      - required
    locations:
      - component: comp-task-form
        role: input
      - component: comp-task-list
        role: output

  - id: dp-task-count
    label: Task Count
    type: number
    source: inferred
    sourceDefinition: Count of all tasks in the system
    constraints:
      - min 0
    locations:
      - component: comp-task-list
        role: output

components:
  - id: comp-task-form
    label: Create Task
    displays: []
    captures:
      - dp-task-title

  - id: comp-task-list
    label: Task List
    displays:
      - dp-task-title
      - dp-task-count
    captures: []

transforms:
  - id: tx-count-tasks
    type: formula
    description: Counts total tasks
    inputs:
      - dp-task-title
    outputs:
      - dp-task-count
    logic:
      type: formula
      content: "task_count = COUNT(tasks)"

dataFlow:
  - from: comp-task-form
    to: dp-task-title
    edgeType: flows-to
    label: form submit
  - from: dp-task-title
    to: tx-count-tasks
    edgeType: transforms
  - from: tx-count-tasks
    to: dp-task-count
    edgeType: derives-from
  - from: dp-task-count
    to: comp-task-list
    edgeType: flows-to
```
