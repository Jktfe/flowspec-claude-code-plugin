# Implementation Walkthrough

A worked example showing how to go from a FlowSpec JSON export to a fully implemented feature.

## The Spec

A simplified task management feature with 3 DataPoints, 1 Component, 1 Transform, and 3 edges:

```json
{
  "version": "1.0.0",
  "metadata": {
    "projectName": "Task Manager",
    "exportedAt": "2026-01-15T10:00:00.000Z",
    "nodeCount": 5,
    "edgeCount": 5
  },
  "dataPoints": [
    {
      "id": "dp-task-title",
      "label": "Task Title",
      "type": "string",
      "source": "captured",
      "sourceDefinition": "User enters task title in creation form",
      "constraints": ["required", "max 200"],
      "locations": [
        { "component": "comp-task-form", "role": "input" },
        { "component": "comp-task-list", "role": "output" }
      ]
    },
    {
      "id": "dp-task-priority",
      "label": "Task Priority",
      "type": "string",
      "source": "captured",
      "sourceDefinition": "User selects priority level from dropdown",
      "constraints": ["required", "enum low/medium/high"],
      "locations": [
        { "component": "comp-task-form", "role": "input" },
        { "component": "comp-task-list", "role": "output" }
      ]
    },
    {
      "id": "dp-task-count",
      "label": "Task Count",
      "type": "number",
      "source": "inferred",
      "sourceDefinition": "Count of tasks grouped by priority",
      "constraints": ["min 0"],
      "locations": [
        { "component": "comp-task-list", "role": "output" }
      ]
    }
  ],
  "components": [
    {
      "id": "comp-task-form",
      "label": "Create Task",
      "displays": [],
      "captures": ["dp-task-title", "dp-task-priority"]
    },
    {
      "id": "comp-task-list",
      "label": "Task List",
      "displays": ["dp-task-title", "dp-task-priority", "dp-task-count"],
      "captures": []
    }
  ],
  "transforms": [
    {
      "id": "tx-count-by-priority",
      "type": "formula",
      "description": "Counts tasks grouped by priority level",
      "inputs": ["dp-task-title", "dp-task-priority"],
      "outputs": ["dp-task-count"],
      "logic": {
        "type": "formula",
        "content": "count_by_priority = GROUP_BY(tasks, priority).COUNT()"
      }
    }
  ],
  "dataFlow": [
    { "from": "comp-task-form", "to": "dp-task-title", "edgeType": "flows-to", "label": "form submit" },
    { "from": "comp-task-form", "to": "dp-task-priority", "edgeType": "flows-to", "label": "form submit" },
    { "from": "dp-task-priority", "to": "tx-count-by-priority", "edgeType": "transforms" },
    { "from": "tx-count-by-priority", "to": "dp-task-count", "edgeType": "derives-from" },
    { "from": "dp-task-count", "to": "comp-task-list", "edgeType": "flows-to" }
  ]
}
```

---

## Step 1: TypeScript Interface

Read `dataPoints` → extract types and group by `locations`:

Both `dp-task-title` and `dp-task-priority` appear in `comp-task-form` (input) and `comp-task-list` (output), so they belong to the same entity.

```typescript
// src/lib/types/task.ts

interface Task {
  id: string;
  title: string;      // dp-task-title: captured, required, max 200
  priority: Priority;  // dp-task-priority: captured, required, enum
}

type Priority = 'low' | 'medium' | 'high';

// dp-task-count is inferred and separate — it's a derived metric, not a Task field
interface TaskCountByPriority {
  priority: Priority;
  count: number;  // dp-task-count: min 0
}
```

**Why `dp-task-count` is separate**: Its `source` is `inferred` and it's produced by a Transform. It's not a property of a single task — it's an aggregate. The spec makes this clear through the `derives-from` edge from `tx-count-by-priority`.

---

## Step 2: Zod Validation Schema

Read `constraints` → map to Zod:

```typescript
// src/lib/schemas/task.ts

import { z } from 'zod';

// Only validate captured DataPoints (form input)
const createTaskSchema = z.object({
  title: z.string()           // dp-task-title
    .min(1, 'Title is required')  // constraint: required
    .max(200),                    // constraint: max 200
  priority: z.enum(['low', 'medium', 'high']),  // constraint: enum
});

type CreateTaskInput = z.infer<typeof createTaskSchema>;
```

**Why no schema for `dp-task-count`**: It's `inferred` — produced by code, not user input. Validating it on input would be wrong. If you need to validate the output of `tx-count-by-priority`, do it inside the transform function.

---

## Step 3: Server Loader (Display Data)

Read `comp-task-list.displays` → determine what to fetch:

```typescript
// src/routes/tasks/+page.server.ts

import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals }) => {
  // comp-task-list displays: dp-task-title, dp-task-priority, dp-task-count
  const tasks = await locals.db.query('SELECT id, title, priority FROM tasks');

  // dp-task-count is inferred via tx-count-by-priority
  const countByPriority = countTasksByPriority(tasks);

  return { tasks, countByPriority };
};
```

**How we knew what to fetch**: `comp-task-list.displays` lists three DataPoints. Two (`dp-task-title`, `dp-task-priority`) are `captured` — they're stored in the database. One (`dp-task-count`) is `inferred` — we need to compute it.

---

## Step 4: Transform Function

Read `tx-count-by-priority` → implement the logic:

```typescript
// src/lib/utils/task-metrics.ts

import type { Task, TaskCountByPriority, Priority } from '$lib/types/task';

// tx-count-by-priority
// logic.content: "count_by_priority = GROUP_BY(tasks, priority).COUNT()"
function countTasksByPriority(tasks: Task[]): TaskCountByPriority[] {
  const counts = new Map<Priority, number>();

  for (const task of tasks) {
    counts.set(task.priority, (counts.get(task.priority) ?? 0) + 1);
  }

  return Array.from(counts.entries()).map(([priority, count]) => ({
    priority,
    count,
  }));
}
```

**How we knew the signature**: `tx-count-by-priority.inputs` lists `dp-task-title` and `dp-task-priority` (both Task fields), and `outputs` lists `dp-task-count`. The logic formula tells us the algorithm.

---

## Step 5: Form Component (Capture Data)

Read `comp-task-form.captures` → create form fields:

```svelte
<!-- src/routes/tasks/CreateTask.svelte -->
<script lang="ts">
  import type { Priority } from '$lib/types/task';

  let title = $state('');
  let priority = $state<Priority>('medium');
</script>

<!-- comp-task-form captures: dp-task-title, dp-task-priority -->
<form method="POST" action="?/create">
  <!-- dp-task-title: string, required, max 200 -->
  <label>
    Task Title
    <input
      name="title"
      bind:value={title}
      required
      maxlength={200}
    />
  </label>

  <!-- dp-task-priority: string, required, enum low/medium/high -->
  <label>
    Priority
    <select name="priority" bind:value={priority} required>
      <option value="low">Low</option>
      <option value="medium">Medium</option>
      <option value="high">High</option>
    </select>
  </label>

  <button type="submit">Create Task</button>
</form>
```

**How we knew the form fields**: `comp-task-form.captures` lists exactly which DataPoints the form collects. We looked up each DataPoint's `type` and `constraints` to set HTML attributes.

---

## Step 6: Form Action (Process Submission)

The edge `comp-task-form →[flows-to]→ dp-task-title` tells us the form stores captured data:

```typescript
// src/routes/tasks/+page.server.ts

import { createTaskSchema } from '$lib/schemas/task';
import { fail } from '@sveltejs/kit';
import type { Actions } from './$types';

export const actions: Actions = {
  create: async ({ request, locals }) => {
    const formData = await request.formData();
    const result = createTaskSchema.safeParse({
      title: formData.get('title'),
      priority: formData.get('priority'),
    });

    if (!result.success) {
      return fail(400, { errors: result.error.flatten().fieldErrors });
    }

    await locals.db.query(
      'INSERT INTO tasks (title, priority) VALUES ($1, $2)',
      [result.data.title, result.data.priority]
    );
  }
};
```

---

## Step 7: Display Component

Wire it all together in the page:

```svelte
<!-- src/routes/tasks/+page.svelte -->
<script lang="ts">
  import CreateTask from './CreateTask.svelte';

  let { data } = $props();
</script>

<h1>Tasks</h1>

<!-- comp-task-list displays dp-task-count (inferred) -->
<div class="priority-counts">
  {#each data.countByPriority as { priority, count }}
    <span>{priority}: {count}</span>
  {/each}
</div>

<!-- comp-task-list displays dp-task-title, dp-task-priority (captured) -->
<ul>
  {#each data.tasks as task}
    <li>
      <span class="priority-{task.priority}">{task.priority}</span>
      {task.title}
    </li>
  {/each}
</ul>

<!-- comp-task-form -->
<CreateTask />
```

---

## Summary: Spec → Code Mapping

| Spec Element | Code Artifact |
|-------------|---------------|
| `dp-task-title`, `dp-task-priority` | `Task` interface fields |
| `dp-task-count` | `TaskCountByPriority` interface |
| Constraints on captured DataPoints | Zod schema + HTML attributes |
| `comp-task-list.displays` | Server loader return data |
| `comp-task-form.captures` | Form fields |
| `tx-count-by-priority` | `countTasksByPriority()` function |
| `flows-to` edges | Form action → DB insert, loader → page |
| `transforms` edge | Function input wiring |
| `derives-from` edge | Function output wiring |

The spec didn't dictate styling, layout, error handling, or UX — those are implementation decisions. But every type, field, validation rule, and data flow came directly from the JSON.
