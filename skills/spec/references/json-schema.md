# FlowSpec JSON Schema Reference

## Version

Current schema version: `1.2.0`

## Top-Level Structure

```json
{
  "version": "1.2.0",
  "metadata": {
    "projectName": "string",        // Name of the project
    "exportedAt": "string",         // ISO 8601 timestamp
    "nodeCount": "number",          // Total DataPoints + Components + Transforms + Tables
    "edgeCount": "number"           // Total edges in dataFlow
  },
  "dataPoints": "DataPoint[]",      // All data elements
  "components": "Component[]",      // All UI components/pages
  "transforms": "Transform[]",      // All business logic nodes
  "tables": "Table[]",              // Data persistence sources (v1.2.0+)
  "dataFlow": "Edge[]",             // All connections between nodes
  "screens": "Screen[]"             // Annotated wireframe screens with regions (v1.1.0+)
}
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

## Table (v1.2.0+)

A data persistence source — database table, API endpoint, or file.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | `string` | Yes | Unique ID. Convention: `tbl-{name}` |
| `label` | `string` | Yes | Human-readable name |
| `sourceType` | `TableSourceType` | Yes | One of: `database`, `api`, `file`, `manual` |
| `columns` | `Column[]` | Yes | Column definitions (can be empty) |
| `endpoint` | `string` | No | API endpoint or connection string |

### Column

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | Column name |
| `type` | `DataType` | One of: `string`, `number`, `boolean`, `object`, `array` |

### TableSourceType Values

| Type | Use For |
|------|---------|
| `database` | SQL/NoSQL database tables |
| `api` | External REST/GraphQL API endpoints |
| `file` | CSV, JSON, or other file-based data sources |
| `manual` | Manually maintained data (spreadsheets, config) |

---

## DataFlow Edge

A connection between two nodes.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | `string` | Yes | Source node ID (any node type) |
| `to` | `string` | Yes | Target node ID (any node type) |
| `edgeType` | `EdgeType` | Yes | One of: `flows-to`, `derives-from`, `transforms`, `validates`, `contains` |
| `label` | `string` | No | Describes the connection |

### EdgeType Values

| Type | Meaning | Typical Connection |
|------|---------|-------------------|
| `flows-to` | Data moves from source to target | DataPoint -> Component, Component -> DataPoint |
| `derives-from` | Target is computed from source | Transform -> inferred DataPoint |
| `transforms` | Data passes through a transform | DataPoint -> Transform |
| `validates` | Transform validates target data | validation Transform -> DataPoint |
| `contains` | Screen/Table membership | Screen -> captured DP, Element -> Screen, Table -> DataPoint |

### Data Flow Direction Rules (CRITICAL — arrows always represent data flow)

| DataPoint Source | Edge Direction | Meaning |
|-----------------|----------------|---------|
| **Captured** | Screen/Component → DataPoint | UI captures user input, data flows to DP |
| **Retrieved** | Table/Transform/Component → DataPoint | Data source feeds the DP |
| **Inferred** | Transform → DataPoint | Computed output flows to DP |
| **Display** | DataPoint → Component/Screen | Data flows to UI for rendering |

**Contains edge direction:** Captured DPs: Screen → DP. All other elements: Element → Screen (data flows to screen for display).

### Connection Rules

| Source -> Target | Allowed | Notes |
|-----------------|---------|-------|
| DataPoint -> DataPoint | Yes | Must have compatible types |
| DataPoint -> Component | Yes | Data flows to UI |
| DataPoint -> Screen | Yes | Non-captured data flows to screen for display |
| DataPoint -> Transform | Yes | Data inputs to logic |
| Transform -> DataPoint | Yes | Computed outputs |
| Transform -> Component | Yes | Direct to display |
| Component -> DataPoint | Yes | Only if target is `captured` |
| Screen -> DataPoint | Yes | Only if target is `captured` |
| Table -> DataPoint | Yes | `contains` edge type (structural) |
| Self-connection | No | |
| Duplicate connection | No | Same source + target |

---

## Screen (v1.1.0+)

An annotated wireframe or screenshot with regions mapping UI areas to data elements.

```json
{
  "screens": [
    {
      "id": "string",                     // Unique screen ID
      "name": "string",                   // Human-readable screen name
      "imageFilename": "string",          // Optional: attached image filename
      "regions": [
        {
          "id": "string",                 // Unique region ID
          "label": "string",             // Optional: region label (e.g. "Login Form")
          "position": {
            "x": "number",              // Top-left X as percentage (0-100)
            "y": "number"               // Top-left Y as percentage (0-100)
          },
          "size": {
            "width": "number",           // Width as percentage (0-100)
            "height": "number"           // Height as percentage (0-100)
          },
          "elements": [
            {
              "nodeId": "string",        // Reference to a DataPoint/Component/Transform ID
              "nodeLabel": "string",     // Label of the referenced node
              "nodeType": "string"       // datapoint | component | transform | table
            }
          ],
          "componentNodeId": "string"    // Optional: promoted Component node ID
        }
      ]
    }
  ]
}
```

### Notes

- Screens are optional — exports without screens are still valid
- Regions use percentage-based coordinates (0-100) relative to the screen image
- Elements reference node IDs from the dataPoints/components/transforms/tables sections
- A node can appear in multiple regions across multiple screens
- `componentNodeId` links a region to a Component node on the canvas (optional)

---

## Complete Example

```json
{
  "version": "1.2.0",
  "metadata": {
    "projectName": "Example App",
    "exportedAt": "2026-02-08T12:00:00.000Z",
    "nodeCount": 7,
    "edgeCount": 6
  },
  "dataPoints": [
    {
      "id": "dp-user-email",
      "label": "User Email",
      "type": "string",
      "source": "captured",
      "sourceDefinition": "User enters email in registration form",
      "constraints": ["required", "email format"],
      "locations": [
        { "component": "comp-register", "role": "input" },
        { "component": "comp-profile", "role": "output" }
      ]
    },
    {
      "id": "dp-account-age",
      "label": "Account Age",
      "type": "number",
      "source": "inferred",
      "sourceDefinition": "Days since registration date",
      "constraints": ["min 0"],
      "locations": [
        { "component": "comp-profile", "role": "output" }
      ]
    }
  ],
  "components": [
    {
      "id": "comp-register",
      "label": "Registration Form",
      "displays": [],
      "captures": ["dp-user-email"]
    },
    {
      "id": "comp-profile",
      "label": "User Profile",
      "displays": ["dp-user-email", "dp-account-age"],
      "captures": []
    }
  ],
  "transforms": [
    {
      "id": "tx-calc-account-age",
      "type": "formula",
      "description": "Calculates days since user registered",
      "inputs": ["dp-user-email"],
      "outputs": ["dp-account-age"],
      "logic": {
        "type": "formula",
        "content": "account_age = (today - registration_date).days"
      }
    }
  ],
  "tables": [
    {
      "id": "tbl-users",
      "label": "Users Table",
      "sourceType": "database",
      "columns": [
        { "name": "email", "type": "string" },
        { "name": "created_at", "type": "string" },
        { "name": "name", "type": "string" }
      ],
      "endpoint": ""
    }
  ],
  "dataFlow": [
    { "from": "comp-register", "to": "dp-user-email", "edgeType": "flows-to", "label": "form submit" },
    { "from": "dp-user-email", "to": "tx-calc-account-age", "edgeType": "transforms" },
    { "from": "tx-calc-account-age", "to": "dp-account-age", "edgeType": "derives-from" },
    { "from": "dp-account-age", "to": "comp-profile", "edgeType": "flows-to" },
    { "from": "tbl-users", "to": "dp-user-email", "edgeType": "contains", "label": "users.email column" },
    { "from": "dp-user-email", "to": "comp-profile", "edgeType": "flows-to", "label": "display in header" }
  ],
  "screens": [
    {
      "id": "screen-profile",
      "name": "Profile Page",
      "imageFilename": "profile-wireframe.png",
      "regions": [
        {
          "id": "region-header",
          "label": "User Header",
          "position": { "x": 5, "y": 2 },
          "size": { "width": 90, "height": 15 },
          "elements": [
            { "nodeId": "dp-user-email", "nodeLabel": "User Email", "nodeType": "datapoint" },
            { "nodeId": "dp-account-age", "nodeLabel": "Account Age", "nodeType": "datapoint" }
          ]
        }
      ]
    }
  ]
}
```
