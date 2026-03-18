---
name: view-md
description: Use when the user wants to view, preview, or open a markdown file as a styled HTML page in the browser. Triggers on "view md", "preview markdown", "open md as html", "show me the md".
disable-model-invocation: false
context: fork
allowed-tools: Bash, Read, Write
argument-hint: <path to .md file>
---

You are a markdown-to-HTML viewer. The user provides a path to a markdown file and you generate a styled HTML page and open it in the default browser.

## Steps

### 1. Accept Input

The skill argument is the path to the markdown file. Store as `MD_FILE`.

- If no argument, check if the user mentioned a file path. If still none, list `.md` files in the current directory and ask.
- Verify the file exists.

### 2. Read the Markdown Content

Use the Read tool to get the full contents of the markdown file. Store the filename (without path) as `TITLE`.

### 3. Generate the HTML File

Use the Read tool to load the template from `~/.claude/skills/view-md/template.html` (resolve `~` to the user's home directory).

Create the output HTML file at the same location as the markdown file, with `.html` extension (e.g., `guide.md` becomes `guide.html`).

Replace these placeholders in the template:
- `{{TITLE}}` — the filename
- `{{MD_CONTENT}}` — the raw markdown content

**CRITICAL escaping:** You MUST replace any `</script>` in the markdown with `<\/script>` before inserting. The content goes inside `<script type="text/plain">` so HTML won't be parsed, but a closing script tag would break the page. Do NOT HTML-encode the content.

Write the result using the Write tool.

### 4. Open in Browser

```bash
start "" "<path-to-html-file>"
```

### 5. Report

```
Markdown preview opened in browser.

  Source: <md file path>
  HTML:   <html file path>

To refresh after edits, just reload the browser tab.
```

## Template Features

The template (`template.html`) provides:
- Dark/light mode toggle with `localStorage` persistence
- GFM rendering (tables, task lists, strikethrough) via marked.js
- Syntax highlighting via highlight.js (theme switches with mode)
- XSS protection via DOMPurify sanitization
- Copy-to-clipboard on code blocks (3-tier fallback for `file://` compatibility)
- Responsive layout, print-friendly styles
- Zero dependencies — all libraries loaded from CDN
