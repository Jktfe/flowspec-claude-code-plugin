# FlowSpec Plugin for Claude Code

A Claude Code plugin that turns [FlowSpec](https://flowspec.dev) YAML exports into your source of truth for spec-driven development. No API or MCP servers required — pure knowledge that works with any codebase.

## What is FlowSpec?

FlowSpec is a visual data architecture tool. You annotate wireframes with data flows, business logic, and constraints on an infinite canvas, then export structured YAML specifications that describe every data element, UI component, transform, and the edges connecting them.

## Skills

### `flowspec:spec` — Spec-Driven Development

Load a FlowSpec YAML export and use it as a requirements document while building features. The spec defines your types, validation rules, data flows, and business logic — Claude references it as you work.

**Use when:**
- Generating TypeScript interfaces and Zod schemas from a spec
- Building pages/components that match the specified data architecture
- Implementing business logic transforms
- Creating API endpoints or database schemas from data points
- Any feature work where a FlowSpec export describes the requirements

**Usage:**
```
/flowspec:spec path/to/flowspec-export.yaml
```

### `flowspec:wireframe` — Screenshot Component Design

Combine a wireframe screenshot with a sub-canvas YAML to implement UI components with both visual and data context. Claude sees the screenshot AND knows what data goes where via placement coordinates.

**Use when:**
- Implementing a specific UI section from a wireframe
- Building components where visual layout and data placement both matter
- Translating annotated wireframes into production code

**Usage:**
```
/flowspec:wireframe path/to/screenshot.png path/to/subcanvas.yaml
```

Optionally include the main spec for full type and constraint information:
```
/flowspec:wireframe path/to/screenshot.png path/to/subcanvas.yaml path/to/main-spec.yaml
```

## Installation

```bash
claude plugin add flowspec
```

Or install locally from a cloned repo:

```bash
claude plugin add --local ./flowspec-claude-code-plugin
```

## How It Works

Both skills are pure knowledge — they teach Claude how to read and use FlowSpec YAML exports. No external services, MCP servers, or API keys are needed.

- **Spec skill**: Reads your YAML, maps DataPoints to types, Components to routes, Transforms to functions, and DataFlow edges to your dependency graph
- **Wireframe skill**: Reads your screenshot (multimodal vision) and sub-canvas YAML (percentage coordinates), then combines visual analysis with data placement to build accurate components

## Example Workflow

1. Design your data architecture in [FlowSpec](https://flowspec.dev)
2. Export the main YAML and any wireframe sub-canvases
3. Start a Claude Code session in your project
4. Load the spec: `/flowspec:spec ./flowspec-export.yaml`
5. Ask Claude to build features: "Generate TypeScript types for the user profile section"
6. For specific UI sections: `/flowspec:wireframe ./profile-header.png ./profile-header.yaml`

## License

MIT
