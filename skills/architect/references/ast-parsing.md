# AST Parsing Strategies for UC2: Codebase → Dataflow

This document provides framework-specific Abstract Syntax Tree (AST) parsing techniques for extracting data architecture from existing codebases.

## Overview

**Goal:** Convert source code into FlowSpec JSON by parsing:
1. **Type definitions** → DataPoints
2. **Routes/Pages** → Components
3. **Utility functions** → Transforms
4. **Data flow** → Edges

**Approach:** Use language-specific AST parsers + pattern recognition.

---

## TypeScript Interface Extraction

TypeScript type definitions are the richest source of DataPoint information.

### Using TypeScript Compiler API

```typescript
import ts from 'typescript';

function extractTypesFromFile(filePath: string): DataPoint[] {
  const program = ts.createProgram([filePath], {});
  const sourceFile = program.getSourceFile(filePath);
  const checker = program.getTypeChecker();
  const dataPoints: DataPoint[] = [];

  function visit(node: ts.Node) {
    // Interface declarations
    if (ts.isInterfaceDeclaration(node)) {
      const interfaceName = node.name.text;

      for (const member of node.members) {
        if (ts.isPropertySignature(member) && member.name) {
          const propertyName = member.name.getText(sourceFile);
          const type = checker.getTypeAtLocation(member);
          const typeName = checker.typeToString(type);

          dataPoints.push({
            id: `${interfaceName.toLowerCase()}_${propertyName}`,
            label: propertyName,
            type: mapTSTypeToFlowSpec(typeName),
            source: 'inferred', // Default, refine based on usage
            sourceDefinition: `Type: ${interfaceName}`,
            constraints: extractConstraints(member)
          });
        }
      }
    }

    // Type aliases
    if (ts.isTypeAliasDeclaration(node)) {
      const aliasName = node.name.text;
      const type = checker.getTypeAtLocation(node);

      // Handle object type literals
      if (type.isClassOrInterface() || type.getProperties().length > 0) {
        for (const prop of type.getProperties()) {
          const propType = checker.getTypeOfSymbolAtLocation(prop, node);
          dataPoints.push({
            id: `${aliasName.toLowerCase()}_${prop.name}`,
            label: prop.name,
            type: mapTSTypeToFlowSpec(checker.typeToString(propType)),
            source: 'inferred',
            sourceDefinition: `Type: ${aliasName}`,
            constraints: []
          });
        }
      }
    }

    ts.forEachChild(node, visit);
  }

  visit(sourceFile!);
  return dataPoints;
}

function mapTSTypeToFlowSpec(tsType: string): DataType {
  if (tsType.includes('string')) return 'string';
  if (tsType.includes('number')) return 'number';
  if (tsType.includes('boolean')) return 'boolean';
  if (tsType.includes('[]') || tsType.includes('Array')) return 'array';
  return 'object';
}

function extractConstraints(member: ts.PropertySignature): string[] {
  const constraints: string[] = [];

  // Check for optional vs required
  if (!member.questionToken) {
    constraints.push('required');
  }

  // Check JSDoc comments for validation hints
  const jsDoc = ts.getJSDocTags(member);
  for (const tag of jsDoc) {
    if (tag.tagName.text === 'example' && tag.comment) {
      // Parse examples for constraints
      const comment = tag.comment.toString();
      if (comment.includes('email')) constraints.push('email-format');
      if (comment.includes('min:')) {
        const match = comment.match(/min:\s*(\d+)/);
        if (match) constraints.push(`min:${match[1]}`);
      }
    }
  }

  return constraints;
}
```

### Using Zod Schemas (SvelteKit/Next.js)

Many projects use Zod for runtime validation. Parse Zod schemas directly:

```typescript
function extractZodSchema(filePath: string): DataPoint[] {
  const source = fs.readFileSync(filePath, 'utf-8');
  const dataPoints: DataPoint[] = [];

  // Regex pattern for z.object({ ... })
  const objectSchemaRegex = /z\.object\(\{([^}]+)\}\)/gs;
  const matches = source.matchAll(objectSchemaRegex);

  for (const match of matches) {
    const schemaBody = match[1];

    // Extract field definitions: fieldName: z.string().email()
    const fieldRegex = /(\w+):\s*z\.(\w+)\(\)([^\n,]*)/g;
    const fields = schemaBody.matchAll(fieldRegex);

    for (const field of fields) {
      const [, name, zodType, validators] = field;

      dataPoints.push({
        id: name.toLowerCase(),
        label: name,
        type: mapZodTypeToFlowSpec(zodType),
        source: 'captured', // Zod schemas typically validate user input
        sourceDefinition: 'Zod schema validation',
        constraints: extractZodConstraints(validators)
      });
    }
  }

  return dataPoints;
}

function mapZodTypeToFlowSpec(zodType: string): DataType {
  const map: Record<string, DataType> = {
    string: 'string',
    number: 'number',
    boolean: 'boolean',
    array: 'array',
    object: 'object'
  };
  return map[zodType] || 'string';
}

function extractZodConstraints(validators: string): string[] {
  const constraints: string[] = [];

  if (validators.includes('.email()')) constraints.push('email-format');
  if (validators.includes('.min(')) {
    const match = validators.match(/\.min\((\d+)\)/);
    if (match) constraints.push(`min:${match[1]}`);
  }
  if (validators.includes('.max(')) {
    const match = validators.match(/\.max\((\d+)\)/);
    if (match) constraints.push(`max:${match[1]}`);
  }
  if (validators.includes('.optional()')) {
    // Do nothing (absence of 'required' implies optional)
  } else {
    constraints.push('required');
  }

  return constraints;
}
```

---

## SvelteKit Route Parsing

SvelteKit routes use `+page.svelte`, `+page.ts`, `+page.server.ts` files.

### Extract Components from Routes

```typescript
import { parse as svelteParse } from 'svelte/compiler';

function extractSvelteComponent(routePath: string): Component {
  const pageFile = fs.readFileSync(`${routePath}/+page.svelte`, 'utf-8');
  const ast = svelteParse(pageFile);

  const component: Component = {
    id: routeToId(routePath),
    label: routeToLabel(routePath),
    displays: [],
    captures: [],
    wireframeRef: undefined
  };

  // Find data usage in template
  // Look for {$dataPoint} or {data.field}
  const dataBindings = findDataBindings(ast.html);
  component.displays = dataBindings.displays;
  component.captures = dataBindings.captures;

  return component;
}

function findDataBindings(html: any): { displays: string[]; captures: string[] } {
  const displays: string[] = [];
  const captures: string[] = [];

  function traverse(node: any) {
    // Text interpolation: {data.field}
    if (node.type === 'MustacheTag' && node.expression) {
      const expr = node.expression;
      if (expr.type === 'MemberExpression') {
        displays.push(expr.property.name);
      }
    }

    // Input bindings: bind:value={field}
    if (node.type === 'Element' && node.name === 'input') {
      for (const attr of node.attributes) {
        if (attr.name === 'bind:value' && attr.expression) {
          captures.push(attr.expression.name);
        }
      }
    }

    // Form actions: <form method="POST">
    if (node.type === 'Element' && node.name === 'form') {
      const methodAttr = node.attributes.find((a: any) => a.name === 'method');
      if (methodAttr && methodAttr.value[0].data === 'POST') {
        // This component captures form data
        // Look for input fields within this form
        const inputs = findInputsInNode(node);
        captures.push(...inputs);
      }
    }

    if (node.children) {
      for (const child of node.children) {
        traverse(child);
      }
    }
  }

  traverse(html);

  return { displays: [...new Set(displays)], captures: [...new Set(captures)] };
}

function findInputsInNode(node: any): string[] {
  const inputs: string[] = [];

  function traverse(n: any) {
    if (n.type === 'Element' && n.name === 'input') {
      const nameAttr = n.attributes.find((a: any) => a.name === 'name');
      if (nameAttr) {
        inputs.push(nameAttr.value[0].data);
      }
    }
    if (n.children) {
      for (const child of n.children) {
        traverse(child);
      }
    }
  }

  traverse(node);
  return inputs;
}
```

### Extract Data Fetching (Load Functions)

```typescript
function extractLoadFunction(routePath: string): DataPoint[] {
  const serverFile = `${routePath}/+page.server.ts`;
  if (!fs.existsSync(serverFile)) return [];

  const source = fs.readFileSync(serverFile, 'utf-8');
  const dataPoints: DataPoint[] = [];

  // Look for export function load() { ... }
  const loadFnRegex = /export\s+(?:async\s+)?function\s+load\s*\([^)]*\)\s*\{([^}]+)\}/s;
  const match = source.match(loadFnRegex);

  if (match) {
    const body = match[1];

    // Find database queries or fetch calls
    const queryRegex = /(?:await\s+)?(?:db|prisma|supabase)\.(\w+)\.(\w+)/g;
    const queries = body.matchAll(queryRegex);

    for (const query of queries) {
      const [, table, operation] = query;

      // Create DataPoint for fetched data
      dataPoints.push({
        id: `${table}_data`,
        label: `${table} Data`,
        type: 'object',
        source: 'inferred',
        sourceDefinition: `Database query: ${table}.${operation}`,
        constraints: []
      });
    }

    // Find return { ... } statement
    const returnRegex = /return\s+\{([^}]+)\}/s;
    const returnMatch = body.match(returnRegex);

    if (returnMatch) {
      const returnBody = returnMatch[1];
      const fields = returnBody.split(',').map(f => f.trim().split(':')[0]);

      for (const field of fields) {
        if (field && !dataPoints.find(dp => dp.id === field)) {
          dataPoints.push({
            id: field,
            label: field,
            type: 'object',
            source: 'inferred',
            sourceDefinition: 'Load function return',
            constraints: []
          });
        }
      }
    }
  }

  return dataPoints;
}
```

### Svelte 5 Runes Detection (Regex Fallback)

Svelte 5 uses `$state`, `$derived`, `$effect` runes. Parse with regex since compiler API may not fully support them yet:

```typescript
function extractSvelte5Runes(componentPath: string): DataPoint[] {
  const source = fs.readFileSync(componentPath, 'utf-8');
  const dataPoints: DataPoint[] = [];

  // $state rune: let count = $state(0)
  const stateRegex = /let\s+(\w+)\s*=\s*\$state\(([^)]+)\)/g;
  const states = source.matchAll(stateRegex);

  for (const state of states) {
    const [, name, initialValue] = state;
    const type = inferTypeFromValue(initialValue);

    dataPoints.push({
      id: name,
      label: name,
      type: type,
      source: 'captured', // $state is usually user-modifiable
      sourceDefinition: 'Svelte 5 $state rune',
      constraints: []
    });
  }

  // $derived rune: let doubled = $derived(count * 2)
  const derivedRegex = /let\s+(\w+)\s*=\s*\$derived\(([^)]+)\)/g;
  const deriveds = source.matchAll(derivedRegex);

  for (const derived of deriveds) {
    const [, name, expression] = derived;

    dataPoints.push({
      id: name,
      label: name,
      type: 'number', // Assume number for calculations
      source: 'inferred',
      sourceDefinition: `Derived: ${expression}`,
      constraints: []
    });
  }

  return dataPoints;
}

function inferTypeFromValue(value: string): DataType {
  if (value.match(/^\d+$/)) return 'number';
  if (value.match(/^['"`]/)) return 'string';
  if (value === 'true' || value === 'false') return 'boolean';
  if (value.startsWith('[')) return 'array';
  if (value.startsWith('{')) return 'object';
  return 'string';
}
```

---

## React/Next.js Parsing

Use Babel parser for React JSX:

```typescript
import { parse as babelParse } from '@babel/parser';
import traverse from '@babel/traverse';

function extractReactComponent(componentPath: string): Component {
  const source = fs.readFileSync(componentPath, 'utf-8');
  const ast = babelParse(source, {
    sourceType: 'module',
    plugins: ['jsx', 'typescript']
  });

  const component: Component = {
    id: pathToId(componentPath),
    label: pathToLabel(componentPath),
    displays: [],
    captures: []
  };

  traverse(ast, {
    // Find useState calls
    CallExpression(path) {
      if (
        path.node.callee.type === 'Identifier' &&
        path.node.callee.name === 'useState'
      ) {
        const parent = path.findParent((p) => p.isVariableDeclarator());
        if (parent && parent.node.id.type === 'ArrayPattern') {
          const stateName = parent.node.id.elements[0].name;
          component.captures.push(stateName);
        }
      }

      // Find fetch/API calls
      if (
        path.node.callee.type === 'Identifier' &&
        path.node.callee.name === 'fetch'
      ) {
        const parent = path.findParent((p) => p.isVariableDeclarator());
        if (parent && parent.node.id.type === 'Identifier') {
          const dataName = parent.node.id.name;
          component.displays.push(dataName);
        }
      }
    },

    // Find JSX attribute bindings: value={data.field}
    JSXExpressionContainer(path) {
      if (path.node.expression.type === 'MemberExpression') {
        const object = path.node.expression.object;
        const property = path.node.expression.property;

        if (object.type === 'Identifier' && property.type === 'Identifier') {
          // Heuristic: if in input, it's captured; otherwise displayed
          const isInput = path.findParent((p) =>
            p.isJSXElement() && p.node.openingElement.name.name === 'input'
          );

          if (isInput) {
            component.captures.push(property.name);
          } else {
            component.displays.push(property.name);
          }
        }
      }
    }
  });

  return component;
}
```

---

## Transform Extraction from Utilities

Scan `lib/utils`, `lib/services`, or similar directories for business logic functions:

```typescript
function extractTransforms(utilsDir: string): Transform[] {
  const transforms: Transform[] = [];
  const files = fs.readdirSync(utilsDir).filter(f => f.endsWith('.ts') || f.endsWith('.js'));

  for (const file of files) {
    const source = fs.readFileSync(path.join(utilsDir, file), 'utf-8');
    const ast = babelParse(source, {
      sourceType: 'module',
      plugins: ['typescript']
    });

    traverse(ast, {
      // Export function declarations
      FunctionDeclaration(path) {
        if (path.node.id && isExported(path)) {
          const transform = buildTransform(path.node, source);
          if (transform) transforms.push(transform);
        }
      },

      // Export const arrow functions
      VariableDeclarator(path) {
        if (
          path.node.id.type === 'Identifier' &&
          path.node.init?.type === 'ArrowFunctionExpression' &&
          isExported(path.parent)
        ) {
          const transform = buildTransform({
            id: path.node.id,
            params: path.node.init.params,
            body: path.node.init.body
          }, source);
          if (transform) transforms.push(transform);
        }
      }
    });
  }

  return transforms;
}

function buildTransform(node: any, source: string): Transform | null {
  const name = node.id.name;

  // Determine type from name heuristics
  let type: LogicType = 'formula';
  if (name.includes('validate') || name.includes('check')) type = 'validation';
  if (name.includes('workflow') || name.includes('process')) type = 'workflow';

  // Extract input parameters
  const inputs = node.params.map((p: any) => p.name);

  // Extract return type or infer outputs
  const outputs = [name + '_result']; // Simplified

  // Extract logic (function body as string)
  const bodyStart = node.body.start;
  const bodyEnd = node.body.end;
  const logic = source.slice(bodyStart, bodyEnd);

  return {
    id: name,
    label: name,
    type: type,
    description: `Function: ${name}`,
    inputs: inputs,
    outputs: outputs,
    logic: {
      type: 'formula',
      content: logic
    }
  };
}

function isExported(path: any): boolean {
  return path.isExportNamedDeclaration() || path.isExportDefaultDeclaration();
}
```

---

## Data Flow Tracing

Trace imports and function calls to generate edges:

```typescript
function traceDataFlow(projectPath: string): Edge[] {
  const edges: Edge[] = [];
  const files = getAllSourceFiles(projectPath);

  for (const file of files) {
    const source = fs.readFileSync(file, 'utf-8');
    const ast = babelParse(source, {
      sourceType: 'module',
      plugins: ['typescript', 'jsx']
    });

    const imports: Map<string, string> = new Map(); // local name → source module

    traverse(ast, {
      // Track imports
      ImportDeclaration(path) {
        const source = path.node.source.value;
        for (const specifier of path.node.specifiers) {
          if (specifier.type === 'ImportSpecifier') {
            imports.set(specifier.local.name, source);
          }
        }
      },

      // Track function calls
      CallExpression(path) {
        if (path.node.callee.type === 'Identifier') {
          const fnName = path.node.callee.name;
          const importSource = imports.get(fnName);

          if (importSource) {
            // This file uses a function from another module
            // Create edge: importSource → currentFile
            edges.push({
              from: moduleToId(importSource),
              to: fileToId(file),
              type: 'transforms',
              label: `calls ${fnName}`
            });
          }
        }
      }
    });
  }

  return edges;
}
```

---

## Framework Detection

Auto-detect framework to choose appropriate parsing strategy:

```typescript
function detectFramework(projectPath: string): string {
  const packageJson = JSON.parse(
    fs.readFileSync(path.join(projectPath, 'package.json'), 'utf-8')
  );

  const deps = {
    ...packageJson.dependencies,
    ...packageJson.devDependencies
  };

  if (deps['@sveltejs/kit']) return 'sveltekit';
  if (deps['next']) return 'nextjs';
  if (deps['@remix-run/node']) return 'remix';
  if (deps['express']) return 'express';
  if (deps['svelte']) return 'svelte';
  if (deps['react']) return 'react';

  // Check directory structure
  if (fs.existsSync(path.join(projectPath, 'src/routes'))) return 'sveltekit';
  if (fs.existsSync(path.join(projectPath, 'app'))) return 'nextjs';
  if (fs.existsSync(path.join(projectPath, 'pages'))) return 'nextjs';

  return 'unknown';
}
```

---

## Best Practices

1. **Start with types first** - TypeScript interfaces give the most reliable information
2. **Use regex as fallback** - When AST parsing is complex, use targeted regex
3. **Heuristic over precision** - For UC2, 80% accuracy is acceptable (user will refine)
4. **Progressive parsing** - Parse one module at a time, show progress
5. **User confirmation loops** - Show extracted data, ask "Is this correct?"
6. **Handle parse errors gracefully** - Skip unparseable files, warn user

---

## Integration with Architect Skill

In the INTERVIEW phase of UC2, use these parsers:

```typescript
// Step 4.2: AST parsing execution
const framework = detectFramework(codebasePath);

switch (framework) {
  case 'sveltekit':
    dataPoints = extractTypesFromFile('src/lib/types/*.ts');
    components = extractSvelteComponents('src/routes/**/+page.svelte');
    transforms = extractTransforms('src/lib/utils');
    break;

  case 'nextjs':
    dataPoints = extractTypesFromFile('types/*.ts');
    components = extractReactComponents('pages/**/*.tsx');
    transforms = extractTransforms('lib/utils');
    break;

  // ... other frameworks
}

// Build mini-specs per module
for (const component of components) {
  const miniSpec = buildMiniSpec({
    dataPoints: filterRelevant(dataPoints, component),
    component: component,
    transforms: filterRelevant(transforms, component)
  });

  // User confirmation
  ASK: "Module '${component.label}' mapped. Correct? [yes/no/refine]"
}
```

---

## Performance Considerations

- **Parallel parsing**: Parse multiple files concurrently
- **Caching**: Cache AST results to avoid re-parsing
- **Incremental**: Parse only changed files if updating existing project
- **Sampling**: For large codebases (>100 files), offer to sample representative files first

---

## Limitations

- **Dynamic code**: Runtime-generated types/components cannot be parsed statically
- **External APIs**: Third-party API schemas must be manually provided or fetched
- **Complex logic**: Non-trivial business logic may be simplified to placeholders
- **Edge cases**: Some language features (decorators, macros) may not parse correctly

**Mitigation:** Always include user confirmation loops to catch inaccuracies.
