# FlowSpec Sub-Canvas YAML Schema

## Version

Current schema version: `1.0.0`

## Overview

A sub-canvas YAML describes data element placements overlaid on a wireframe screenshot. It is exported from FlowSpec alongside the screenshot image and is used to map data elements to their visual positions on screen.

The `kind: 'sub-canvas'` field distinguishes this from a main FlowSpec YAML export.

## Top-Level Structure

```yaml
version: "1.0.0"
kind: sub-canvas
metadata:
  name: string                # Name of the sub-canvas (e.g., "Dashboard Header")
  screenshotFilename: string  # Filename of the accompanying screenshot image
  screenshotWidth: number     # Original screenshot width in pixels
  screenshotHeight: number    # Original screenshot height in pixels
  placementCount: number      # Number of data elements placed on this screenshot
  exportedAt: string          # ISO 8601 timestamp
  projectName: string         # Parent project name

placements: Placement[]       # Data elements positioned on the screenshot
```

---

## Placement

A single data element positioned on the wireframe screenshot.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sourceNodeId` | `string` | Yes | ID of the node in the main FlowSpec spec (e.g., `dp-user-email`) |
| `sourceNodeLabel` | `string` | Yes | Human-readable label (e.g., "User Email") |
| `sourceNodeType` | `string` | Yes | Node type: `datapoint`, `component`, or `transform` |
| `position` | `Position` | Yes | Percentage-based position on the screenshot |

### Position

| Field | Type | Description |
|-------|------|-------------|
| `x` | `number` | Horizontal position as percentage (0 = left edge, 100 = right edge) |
| `y` | `number` | Vertical position as percentage (0 = top edge, 100 = bottom edge) |

### Important Notes on Position

- Positions are **percentages**, not pixels
- Values range from `0` to `100` (may slightly exceed for elements near edges)
- `(0, 0)` = top-left corner of the screenshot
- `(100, 100)` = bottom-right corner of the screenshot
- Values are rounded to 1 decimal place
- Use proximity to group elements into logical sections (elements within ~5% of each other are likely in the same UI group)

### sourceNodeType Values

| Type | Meaning | What It Tells You |
|------|---------|-------------------|
| `datapoint` | A data element | This position shows where data appears or is entered on screen |
| `component` | A UI component reference | This position marks a sub-component boundary on the wireframe |
| `transform` | A logic node | This position shows where a calculation result appears on screen |

---

## Cross-Referencing the Main Spec

Use `sourceNodeId` to look up full details in the main FlowSpec YAML:

- For `datapoint` nodes: find the matching entry in `dataPoints[]` to get `type`, `source`, `constraints`, etc.
- For `component` nodes: find the matching entry in `components[]` to get `displays`, `captures`
- For `transform` nodes: find the matching entry in `transforms[]` to get `inputs`, `outputs`, `logic`

If the main spec is not available, `sourceNodeLabel` and `sourceNodeType` provide enough context for basic implementation.

---

## Complete Example

```yaml
version: "1.0.0"
kind: sub-canvas
metadata:
  name: User Profile Header
  screenshotFilename: user-profile-header.png
  screenshotWidth: 1440
  screenshotHeight: 900
  placementCount: 5
  exportedAt: "2026-01-20T14:30:00.000Z"
  projectName: My App

placements:
  - sourceNodeId: dp-user-avatar
    sourceNodeLabel: User Avatar
    sourceNodeType: datapoint
    position:
      x: 5.2
      y: 8.1

  - sourceNodeId: dp-user-name
    sourceNodeLabel: User Name
    sourceNodeType: datapoint
    position:
      x: 15.0
      y: 6.3

  - sourceNodeId: dp-user-email
    sourceNodeLabel: User Email
    sourceNodeType: datapoint
    position:
      x: 15.0
      y: 12.5

  - sourceNodeId: dp-account-age
    sourceNodeLabel: Account Age
    sourceNodeType: datapoint
    position:
      x: 85.0
      y: 8.0

  - sourceNodeId: tx-calc-account-age
    sourceNodeLabel: Calculate Account Age
    sourceNodeType: transform
    position:
      x: 85.0
      y: 15.0
```

### Reading This Example

- **User Avatar** (5.2%, 8.1%) — top-left area, likely a profile picture
- **User Name** (15%, 6.3%) — next to the avatar, slightly higher — probably the main heading
- **User Email** (15%, 12.5%) — below the name — a secondary text line
- **Account Age** (85%, 8%) — far right, top — a stat/badge in the header
- **Calculate Account Age** (85%, 15%) — the transform that produces the account age value, positioned near where it displays

Elements at similar x-coordinates (User Name and User Email at 15%) form a vertical group. Elements at similar y-coordinates (User Name and Account Age near 8%) form a horizontal row.
