# FlowSpec Plugin for Claude Code

A Claude Code plugin that turns [FlowSpec](https://flowspec.dev) JSON exports into your source of truth for spec-driven development. No API or MCP servers required — pure knowledge that works with any codebase.

## What is FlowSpec?

FlowSpec is a visual data architecture tool. You annotate wireframes with data flows, business logic, and constraints on an infinite canvas, then export structured JSON specifications that describe every data element, UI component, transform, and the edges connecting them.

## Skills

### `flowspec:spec` — Spec-Driven Development

Load a FlowSpec JSON export and use it as a requirements document while building features. The spec defines your types, validation rules, data flows, and business logic — Claude references it as you work.

**Use when:**
- Generating TypeScript interfaces and Zod schemas from a spec
- Building pages/components that match the specified data architecture
- Implementing business logic transforms
- Creating API endpoints or database schemas from data points
- Any feature work where a FlowSpec export describes the requirements

**Usage:**
```
/flowspec:spec path/to/flowspec-export.json
```

### `flowspec:wireframe` — Screenshot Component Design

Combine a wireframe screenshot with a sub-canvas JSON spec to implement UI components with both visual and data context. Claude sees the screenshot AND knows what data goes where via placement coordinates.

**Use when:**
- Implementing a specific UI section from a wireframe
- Building components where visual layout and data placement both matter
- Translating annotated wireframes into production code

**Usage:**
```
/flowspec:wireframe path/to/screenshot.png path/to/subcanvas.json
```

Optionally include the main spec for full type and constraint information:
```
/flowspec:wireframe path/to/screenshot.png path/to/subcanvas.json path/to/main-spec.json
```

## Installation

### From Marketplace (recommended)

If you have [FlowSpec Desktop](https://flowspec.dev/download) installed, the marketplace is auto-configured during setup:

```bash
claude plugin install flowspec@flowspec-plugins
```

### Direct Installation

Without FlowSpec Desktop, install directly from GitHub:

```bash
claude plugin install flowspec@https://github.com/Jktfe/flowspec-claude-code-plugin
```

Note: Direct GitHub installation may not work due to SSH/authentication issues. Use the marketplace method or local install instead.

### Local Development

Clone the repo and load it with `--plugin-dir`:

```bash
git clone https://github.com/Jktfe/flowspec-claude-code-plugin.git
claude --plugin-dir ./flowspec-claude-code-plugin
```

This loads the plugin for that session. To always load it, add the flag to your shell alias:
### From GitHub (recommended)

```bash
claude plugin install flowspec@https://github.com/Jktfe/flowspec-claude-code-plugin
```

### Local install

Clone the repo and load it with `--plugin-dir`:

```bash
git clone https://github.com/Jktfe/flowspec-claude-code-plugin.git
claude --plugin-dir ./flowspec-claude-code-plugin
```

This loads the plugin for that session. To always load it, add the flag to your shell alias:

```bash
# In your .zshrc or .bashrc
alias claude='claude --plugin-dir /path/to/flowspec-claude-code-plugin'
```

## How It Works

Both skills are pure knowledge — they teach Claude how to read and use FlowSpec JSON exports. No external services, MCP servers, or API keys are needed.

- **Spec skill**: Reads your JSON spec, maps DataPoints to types, Components to routes, Transforms to functions, and DataFlow edges to your dependency graph
- **Wireframe skill**: Reads your screenshot (multimodal vision) and sub-canvas JSON (percentage coordinates), then combines visual analysis with data placement to build accurate components

## Example Workflow

1. Design your data architecture in [FlowSpec](https://flowspec.dev)
2. Export the main JSON spec and any wireframe sub-canvases
3. Start a Claude Code session with the plugin loaded (see Installation above)
4. Load the spec: `/flowspec:spec ./flowspec-export.json`
5. Ask Claude to build features: "Generate TypeScript types for the user profile section"
6. For specific UI sections: `/flowspec:wireframe ./profile-header.png ./profile-header.json`

## License

MIT
