---
name: wireframe
description: >-
  Use when implementing a UI component from a wireframe screenshot and FlowSpec
  sub-canvas YAML. Combines visual analysis of the screenshot with placement
  coordinates to understand what data appears where on screen.
argument_hint: path/to/screenshot.png path/to/subcanvas.yaml [path/to/main-spec.yaml]
---

# FlowSpec Wireframe Component Design

## What This Skill Does

A FlowSpec wireframe export consists of two files:

1. **A screenshot image** — a wireframe or UI mockup captured from the FlowSpec canvas
2. **A sub-canvas YAML** — structured data listing which data elements are placed where on the screenshot, using percentage-based coordinates

Together, they give you both **visual context** (what it looks like) and **data context** (what information goes where). This skill teaches you to use both to implement accurate UI components.

Optionally, a **main FlowSpec YAML** can be provided alongside. When available, it enriches each placement with full type information, constraints, and data flow context.

---

## Step 1: Load the Files

### Required Files

1. **Screenshot image**: Read it with the Read tool (Claude can view images). Analyse the visual layout, hierarchy, spacing, and patterns.
2. **Sub-canvas YAML**: Read it with the Read tool and parse the placements.

### Optional File

3. **Main spec YAML**: If provided, load it to cross-reference `sourceNodeId` values for full type and constraint data.

### Confirm Loading

Summarise what you've loaded:
```
Loaded wireframe: {metadata.name}
- Screenshot: {screenshotFilename} ({screenshotWidth}x{screenshotHeight})
- {placementCount} data elements placed
- Main spec: {loaded or not available}
```

### Loading the Schema Reference

For detailed field definitions, read the [Sub-Canvas Schema Reference](references/subcanvas-schema.md).

---

## Step 2: Analyse the Screenshot

Look at the screenshot image and identify:

1. **Overall layout** — Is it a header, card, table, form, dashboard, sidebar, modal?
2. **Visual hierarchy** — What's prominent? What's secondary? What's subtle?
3. **Grouping** — What elements are visually grouped together (cards, sections, rows)?
4. **Patterns** — Is there repetition (list items, grid cells, table rows)?
5. **Interactive elements** — Buttons, inputs, links, toggles visible in the wireframe?
6. **Whitespace and spacing** — How dense is the layout? What rhythm does it follow?

This visual analysis establishes the **structural intent** of the UI before you overlay data.

---

## Step 3: Map Placements to the Visual

Now overlay the sub-canvas YAML placements onto your visual understanding.

### Reading Placement Coordinates

Positions are **percentages** of the screenshot dimensions:
- `x: 0` = left edge, `x: 100` = right edge
- `y: 0` = top edge, `y: 100` = bottom edge

### Grouping Strategy

1. **Group by spatial proximity** — Elements within ~5% of each other in both x and y likely belong to the same visual group
2. **Group by horizontal bands** — Elements at similar y-coordinates form a row
3. **Group by vertical columns** — Elements at similar x-coordinates form a column
4. **Group by sourceNodeType** — DataPoints in one area, transforms in another can indicate data vs. computed sections

### Example Analysis

Given placements:
```yaml
- sourceNodeLabel: "User Name"      # position: x:15, y:6
- sourceNodeLabel: "User Email"     # position: x:15, y:12
- sourceNodeLabel: "User Avatar"    # position: x:5, y:8
- sourceNodeLabel: "Account Age"    # position: x:85, y:8
```

Interpretation:
- **Left group** (x:5-15): Avatar + Name + Email — this is a user identity block
- **Right group** (x:85): Account Age — this is a stat/badge, separated from the identity block
- **Vertical stacking** (y:6 → y:12 at x:15): Name above Email — text hierarchy

### Understanding sourceNodeType

| Type | What It Tells You About This Position |
|------|--------------------------------------|
| `datapoint` | A data value is displayed or captured here |
| `component` | A sub-component boundary — this area is a distinct UI component |
| `transform` | A computed value is shown here — the result of business logic |

---

## Step 4: Cross-Reference the Main Spec

If the main FlowSpec YAML is available, use `sourceNodeId` to look up full details:

### For DataPoints (sourceNodeType: "datapoint")

Find `sourceNodeId` in `dataPoints[]`:
- **type** → TypeScript type for this data (`string`, `number`, `boolean`, etc.)
- **source** → `captured` means this is an input field; `inferred` means display-only
- **constraints** → Validation rules to enforce
- **sourceDefinition** → How this data enters the system

### For Components (sourceNodeType: "component")

Find `sourceNodeId` in `components[]`:
- **displays** → What data this sub-section shows
- **captures** → What data this sub-section collects

### For Transforms (sourceNodeType: "transform")

Find `sourceNodeId` in `transforms[]`:
- **logic** → The algorithm that produces this value
- **inputs/outputs** → What data feeds in and comes out

### Without the Main Spec

If no main spec is available, you still have:
- `sourceNodeLabel` — the human-readable name tells you what the data IS
- `sourceNodeType` — tells you whether it's data, a sub-component, or a computed value
- `position` — tells you WHERE it goes on screen

This is enough for a reasonable implementation. You just won't have types, constraints, or the full data flow graph.

---

## Step 5: Implement the Component

With both visual and data context, build the component:

### Layout Structure

1. Use the screenshot's visual structure to determine your HTML/component hierarchy
2. Use placement groups to define sections, rows, and columns
3. Match the visual density and spacing you see in the screenshot

### Data Binding

For each placement:

1. **If sourceNodeType is `datapoint` and source is `captured`**: Create an input field at this position
2. **If sourceNodeType is `datapoint` and source is `inferred`**: Display the value read-only at this position
3. **If sourceNodeType is `transform`**: Display the computed result at this position
4. **If sourceNodeType is `component`**: This marks a sub-component boundary — consider extracting it

### Validation

If you have the main spec:
- Apply constraints from matched DataPoints to inputs
- Use `source: captured` DataPoints to determine which fields are editable
- Use `source: inferred` DataPoints to determine which are display-only

### Styling Guidance

The screenshot IS the design spec. Match:
- Visual hierarchy (headings, body text, captions)
- Grouping and spacing
- The general layout structure (grid, flexbox, positioning)
- Relative proportions between sections

Don't try to pixel-match the screenshot — use it as a structural guide. The percentages tell you relative positioning, not exact CSS values.

---

## Common Mistakes

### 1. Ignoring the screenshot

The screenshot provides visual context that the YAML alone cannot. Don't just read the YAML and ignore the image. The screenshot shows:
- Visual hierarchy and emphasis
- Whitespace and breathing room
- The intended "feel" of the layout

### 2. Treating percentages as pixels

Positions are percentages of the screenshot dimensions, NOT pixel values. `x: 50, y: 50` means the CENTER of the screenshot, not 50px from the top-left.

### 3. Ignoring visual grouping

Elements that are visually grouped in the screenshot should be grouped in your component's DOM structure, even if the YAML lists them as separate placements.

### 4. Making inferred data editable

If `sourceNodeType` is `datapoint` and the main spec shows `source: inferred`, this value is COMPUTED. Display it read-only. If there's no main spec but the `sourceNodeLabel` contains words like "total", "count", "average", "calculated", or "estimated", it's likely inferred.

### 5. Forgetting responsive design

The screenshot shows ONE viewport size. Your implementation should adapt to different screen sizes. Use the placement positions as a guide for relative layout, not as fixed positions.

### 6. Placing everything absolutely

Percentage coordinates describe WHERE data appears on the wireframe, not HOW to position it in CSS. Use semantic layout (flexbox, grid, logical flow) to achieve the same visual arrangement.

---

## Quick Reference

```
Screenshot        → Visual structure, hierarchy, grouping, spacing
Sub-canvas YAML   → What data goes where (percentage coordinates)
Main spec (opt)   → Types, constraints, captured vs inferred, data flow

position.x: 0-100  → left to right (percentage)
position.y: 0-100  → top to bottom (percentage)

sourceNodeType: datapoint  → data value at this position
sourceNodeType: component  → sub-component boundary
sourceNodeType: transform  → computed value at this position
```

For the complete sub-canvas schema, see [references/subcanvas-schema.md](references/subcanvas-schema.md).
