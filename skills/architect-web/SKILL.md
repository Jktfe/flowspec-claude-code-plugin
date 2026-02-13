---
name: architect-web
description: >-
  Create and manage FlowSpec data architecture projects via the web app at
  flowspec.app using Chrome browser tools. For users without the desktop app.
  Generates JSON locally and uses light browser automation to import into
  the web editor. Requires the Claude in Chrome extension.
argument_hint: "[wireframe images | codebase path | 'new project']"
---

# FlowSpec Architect (Web / Chrome)

Same three workflows as `architect`, but uses Chrome browser tools instead of MCP — for users without the desktop app.

**Core pattern:** Generate JSON locally in Claude Code -> inject into flowspec.app via Chrome tools -> iterate.

**Requires:** Claude in Chrome extension (`mcp__claude-in-chrome__*` tools).

---

## Prerequisites & Auth Check

1. Call `mcp__claude-in-chrome__tabs_context_mcp` to find an existing flowspec.app tab.
2. If no tab exists:
   ```
   mcp__claude-in-chrome__tabs_create_mcp(url: "https://flowspec.app/projects")
   ```
   Navigate to https://flowspec.app/projects to browse and create projects from the dashboard, or go directly to https://flowspec.app/editor to open the canvas.
3. Use `mcp__claude-in-chrome__read_page` to check for auth state:
   - Look for `.signout-btn` or user email display to confirm signed-in state
   - If not signed in: instruct user to sign in manually at flowspec.app/sign-in
4. Wait for the dashboard or editor to fully load before proceeding.

---

## Interaction Model

| Action | Method |
|--------|--------|
| Generate JSON | Locally in Claude Code (same logic as `architect`) |
| Import JSON into project | `javascript_tool` with synthetic file injection (see [chrome-recipes.md](references/chrome-recipes.md)) |
| Export JSON from project | Click Export button via `javascript_tool`, then `read_page` |
| Create new project | Click "New Project" button via browser tools |
| Upload wireframe images | **User does manually** (browser file picker security) |
| Annotate screen regions | **User does manually** (drag-to-create is more ergonomic) |
| Auto-layout | Click toolbar button via browser tools |

**Fallback:** If any automation step fails, always provide manual instructions so the user can complete the step themselves.

---

## UC1: Wireframe -> Dataflow (Web)

### Step 1: Analyse wireframe images
Use Claude vision to examine wireframe images the user provides:
- Identify UI regions, data elements, forms, displays
- Note interactive elements and data relationships

### Step 2: Interview user
Ask about data types, sources, business logic, constraints — same as `architect` UC1 Step 4.

### Step 3: Build JSON spec
Assemble the JSON following the [v1.2.0 schema](../architect/references/json-schema.md).

### Step 4: Navigate to flowspec.app
Ensure we have a flowspec.app tab open (preferably at `/projects` for the dashboard):
```
mcp__claude-in-chrome__tabs_context_mcp
```
If needed, navigate to https://flowspec.app/projects and create a new project from the dashboard, or go directly to https://flowspec.app/editor.

### Step 5: Import JSON
Use the synthetic file injection recipe from [chrome-recipes.md](references/chrome-recipes.md) to inject the JSON into the web app's import handler.

### Step 6: Guide wireframe upload
Instruct the user to:
1. Open the Screens section in the sidebar
2. Click "Create Screen" or use the image dropzone
3. Upload their wireframe image files
4. Name each screen

### Step 7: Guide region annotation
Instruct the user to:
1. Open a screen in the editor
2. Draw regions over the wireframe by clicking and dragging
3. Name each region and link data elements using the region panel

### Step 8: Export and iterate
Use browser tools to click "Export JSON" and read the result. Review and refine the spec, then re-import if needed.

---

## UC2: Codebase -> Dataflow (Web)

### Steps 1-7: Analyse locally
All codebase analysis happens locally in Claude Code — identical to `architect` UC2 Steps 1-7. See the architect skill's [codebase-analysis.md](references/codebase-analysis.md) methodology (which is the same reference used by the MCP-powered skill).

### Step 8: Create project and import
Navigate to https://flowspec.app/projects, create a new project from the dashboard, and import the JSON via Chrome tools.

### Step 9: Guide auto-layout
Instruct the user to click the Auto Layout button in the toolbar, or use `javascript_tool` to trigger it.

### Step 10: Export and validate
Export JSON via browser tools and validate the spec locally.

---

## UC3: Design New Screens (Web)

### Step 1: Obtain screen image
User designs in an external tool and provides screenshots, or starts with an empty screen.

### Step 2: Guide screen creation
Instruct the user to create screens and upload images via the web UI.

### Step 3: Build JSON for data architecture
Create dataPoints, components, transforms for the screen's data needs.

### Step 4: Import JSON
Import via Chrome tools using the synthetic file injection recipe.

### Step 5: Guide region annotation
Same as UC1 Step 7 — user annotates regions manually in the web UI.

---

## Limitations

- **Cannot automate authentication** — user must sign in manually
- **Image upload requires user interaction** — browser file picker security prevents automation
- **Region annotation is best done manually** — the web UI's drag-to-create is more ergonomic than coordinate-based automation
- **Large JSON files** may need manual file import as fallback — copy JSON to clipboard, user pastes or uses file dialog
- **DOM selectors may change** — if the web app UI updates, the Chrome recipes in `references/chrome-recipes.md` need updating

---

## Chrome Tool Reference

| Tool | Purpose |
|------|---------|
| `tabs_context_mcp` | Get current browser tab info |
| `tabs_create_mcp` | Open new tab |
| `read_page` | Read page content/DOM |
| `javascript_tool` | Execute JavaScript in page |
| `navigate` | Navigate to URL |
| `find` | Find elements on page |
| `form_input` | Fill form fields |
| `computer` | Click/interact with page |

See [chrome-recipes.md](references/chrome-recipes.md) for concrete JavaScript snippets.
