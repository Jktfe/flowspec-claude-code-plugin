# Chrome Recipes for FlowSpec Web App

Concrete JavaScript snippets for interacting with flowspec.app via `mcp__claude-in-chrome__javascript_tool`.

---

## Check Auth State

Verify the user is signed in by checking for the sign-out button:

```javascript
// Returns true if signed in, false otherwise
const signoutBtn = document.querySelector('.signout-btn');
const isSignedIn = !!signoutBtn;
JSON.stringify({ isSignedIn });
```

---

## Read Current Project ID

Parse the project ID from the URL query string:

```javascript
const params = new URLSearchParams(window.location.search);
const projectId = params.get('project');
JSON.stringify({ projectId, url: window.location.href });
```

---

## Navigate to a Project

```javascript
window.location.href = 'https://flowspec.app/editor?project=PROJECT_ID_HERE';
```

---

## Create New Project

Click the "New Project" button in the sidebar:

```javascript
const newBtn = document.querySelector('button');
const buttons = Array.from(document.querySelectorAll('button'));
const newProjectBtn = buttons.find(b => b.textContent.trim().includes('New Project'));
if (newProjectBtn) {
  newProjectBtn.click();
  'Clicked New Project button';
} else {
  'New Project button not found — available buttons: ' + buttons.map(b => b.textContent.trim()).slice(0, 10).join(', ');
}
```

---

## Import JSON via Synthetic File Injection

This is the core recipe — creates a synthetic `File` object and injects it into the hidden file input used by the "Import JSON" button.

**DOM target:** `label.import-btn > input[type="file"]` (from `Sidebar.svelte:405-419`)

```javascript
const jsonContent = `JSON_CONTENT_HERE`;

// Find the hidden file input inside the Import JSON label
const fileInput = document.querySelector('.import-btn input[type="file"]');
if (!fileInput) {
  'Import file input not found — ensure the sidebar is visible';
} else {
  // Create a synthetic File from the JSON string
  const blob = new Blob([jsonContent], { type: 'application/json' });
  const file = new File([blob], 'import.json', { type: 'application/json' });

  // Create a synthetic FileList via DataTransfer
  const dt = new DataTransfer();
  dt.items.add(file);
  fileInput.files = dt.files;

  // Dispatch change event to trigger the onchange handler
  fileInput.dispatchEvent(new Event('change', { bubbles: true }));

  'JSON imported successfully (' + jsonContent.length + ' chars)';
}
```

**Important:** Replace `JSON_CONTENT_HERE` with the actual JSON string. Escape backticks and `${` sequences if present in the JSON.

---

## Export JSON

Click the "Export JSON" button. The export triggers a file download, so we need to intercept it:

```javascript
// Override the default download to capture the JSON content
const buttons = Array.from(document.querySelectorAll('button'));
const exportBtn = buttons.find(b => b.textContent.trim().includes('Export JSON'));
if (exportBtn) {
  exportBtn.click();
  'Clicked Export JSON — check for downloaded file or use read_page to capture content';
} else {
  'Export JSON button not found';
}
```

**Alternative:** After clicking export, use `mcp__claude-in-chrome__read_page` to read any modal or text content that appears.

---

## Click Auto Layout

Trigger the auto-layout button in the toolbar:

```javascript
const buttons = Array.from(document.querySelectorAll('button'));
const layoutBtn = buttons.find(b =>
  b.textContent.trim().includes('Auto Layout') ||
  b.title?.includes('Auto Layout') ||
  b.getAttribute('aria-label')?.includes('layout')
);
if (layoutBtn) {
  layoutBtn.click();
  'Auto layout triggered';
} else {
  'Auto Layout button not found';
}
```

---

## Open Projects List

```javascript
const buttons = Array.from(document.querySelectorAll('button'));
const projectsBtn = buttons.find(b => b.textContent.trim().includes('Projects'));
if (projectsBtn) {
  projectsBtn.click();
  'Opened projects list';
} else {
  'Projects button not found';
}
```

---

## Read Project Name

```javascript
// The project name is in an editable input in the sidebar
const nameInput = document.querySelector('.project-name-input, input[placeholder*="project"], input[placeholder*="Project"]');
if (nameInput) {
  JSON.stringify({ projectName: nameInput.value });
} else {
  // Fallback: look for heading text
  const heading = document.querySelector('h2, .project-name');
  JSON.stringify({ projectName: heading?.textContent?.trim() || 'unknown' });
}
```

---

## Notes

- **DOM selectors are based on the current Sidebar.svelte implementation.** If the web app UI changes, these selectors may need updating.
- **The synthetic file injection bypasses the native file picker**, which cannot be triggered programmatically due to browser security.
- **Large JSON files** may cause issues when injected as inline JavaScript strings. If this happens, instruct the user to manually import via the file dialog instead.
- **Always verify the sidebar is visible** before attempting to interact with its elements. The sidebar is the primary control panel on the left side of the editor.
