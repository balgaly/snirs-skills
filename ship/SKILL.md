---
name: ship
description: Complete workflow to ship changes - commits, reviews, creates interactive HTML report, fixes issues iteratively, runs tests, performs integration testing and full QA with Playwright, generates QA report, and final commit. Use when ready to ship code changes.
disable-model-invocation: true
context: fork
allowed-tools: Read, Write, Edit, Grep, Glob, Bash, mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_click, mcp__playwright__browser_type, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_evaluate, mcp__playwright__browser_console_messages, mcp__playwright__browser_fill_form, mcp__playwright__browser_press_key, mcp__playwright__browser_handle_dialog, mcp__playwright__browser_wait_for, mcp__playwright__browser_close, mcp__playwright__browser_resize
argument-hint: [optional commit message]
---

You are automating the complete code shipping workflow.

## Workflow Steps

### Phase 1: Initial Commit & Review

1. **Check git status** to see what files have changed
2. **Commit current changes** with message:
   - Use user's message if provided via $ARGUMENTS
   - Otherwise use: "WIP: Changes ready for review"
3. **Run code review** using the code-reviewer skill via `/code-reviewer`
4. **Parse review results** and categorize issues by priority

### Phase 2: Interactive Review Report

Create an HTML file at `code-review-report.html` using the Anthropic web designer aesthetic (CSS custom properties, indigo gradient header with radial overlays, subtle shadows, system font stack, pill badges with status dots, monospace code blocks on light backgrounds, uppercase tracking labels, `.btn-primary`/`.btn-secondary` button styles, responsive grid).

The HTML structure must include:
- `.header` with gradient, title, subtitle, and `[TIMESTAMP]` build stamp
- `.meta-bar` below the header: a horizontal bar (dark background, e.g. `var(--gray-800)`) displaying key metadata in a flex row with pill-style items: **Repository** (org/repo name from `git remote`), **Branch** (current branch name), **Generated** (date and time, e.g. "2 Mar 2026, 23:00"), **Commit** (short SHA of HEAD). Each item has a small uppercase label and value. Use `var(--font-mono)` for values.
- `.stats` grid with 3 `.stat-card` elements (critical/warning/suggestion counts, computed dynamically from the issues array)
- `.issues-list` container populated by `renderIssues()` — each issue gets a `.issue` card with checkbox, `.issue-priority` pill badge, title, and expandable details (location, problem, impact, suggested fix in `<pre class="fix-block">`)
- `.actions-panel` with "Generate Fix Instructions" button, readonly textarea, "Copy to Clipboard" button, and `.copy-notice` div
- JavaScript: `renderStats()`, `renderIssues()`, `generateInstructions()`, `copyToClipboard()` with 3-tier clipboard fallback (Clipboard API -> execCommand -> manual select), `fallbackCopy()`, `showCopyNotice()`
- Critical issues are pre-checked; all others unchecked by default

Data placeholder: `const issues = [ISSUES_JSON];`

If `code-review-report.html` already exists in the project root, use it as the canonical reference implementation — replicate its exact CSS and HTML structure, replacing only the data (`[ISSUES_JSON]`) and build stamp (`[TIMESTAMP]`). Otherwise, generate the HTML from the description above.

**Important:** Replace placeholders with actual review data:
- `[ISSUES_JSON]` - JSON array of issues (stats are computed dynamically from this array)
- `[TIMESTAMP]` - Current date/time string (e.g., `v4-20260220-143022`) so users can verify they see the latest version and not a cached page
- `[REPO_NAME]` - Repository name from `git remote get-url origin` (e.g., `balgaly/guessit`)
- `[BRANCH_NAME]` - Current branch from `git branch --show-current`
- `[GENERATED_DATE]` - Human-readable date/time (e.g., `2 Mar 2026, 23:00`)
- `[COMMIT_SHA]` - Short SHA of HEAD from `git rev-parse --short HEAD`

5. **Open the HTML file** in browser automatically
6. **Wait for user input** - tell user to:
   - Review the HTML report
   - Select which issues to fix
   - Click "Generate Fix Instructions"
   - Copy the generated text
   - Paste it back to Claude Code

### Phase 3: Fix Issues Loop

When user pastes back instructions starting with `SHIP_WORKFLOW_FIX_ITEMS:`:

1. **Parse the fix items** from the pasted text
2. **Fix each issue** by:
   - Reading the affected file
   - Applying the fix using Edit tool
   - Verify the fix was applied
3. **Rerun code review** via `/code-reviewer`
4. **If new issues found:**
   - Create new HTML report
   - Repeat Phase 2 and 3
5. **If no issues or user pastes `SHIP_WORKFLOW_CONTINUE`:**
   - Proceed to Phase 4

### Phase 4: Unit Testing

1. **Detect and run tests**:
   - Look for test files (*.test.js, *.spec.js, *.test.ts, etc.)
   - Run them with the appropriate runner (node, jest, vitest, etc.)
2. **If tests pass:** Proceed to Phase 5
3. **If tests fail:**
   - Identify failing tests
   - Fix the issues
   - Rerun tests
   - Repeat until all pass
   - Do NOT tell user about test failures unless you cannot fix them

### Phase 5: Integration Testing with Playwright

Use Playwright MCP tools to test the application end-to-end in a real browser.

**Before starting:** Check if Playwright MCP tools are available by attempting `browser_navigate`. If the tool is not available, skip Phases 5-7 with a warning message and proceed directly to Phase 9 (Final Commit).

#### Approach

1. Identify the main HTML pages in the project
2. Navigate to each using `file://` URLs with absolute paths
3. Test interactive elements (forms, buttons, links, modals)
4. Use `browser_evaluate` to check DOM state, CSS classes, and element properties
5. Use `browser_type` with `slowly: true` for fields needing input event triggers
6. Take screenshots of key states
7. Navigate fresh between independent test groups for clean state

#### Result Tracking

Track each test result as:
```json
{ "name": "test name", "category": "integration", "passed": true/false, "screenshot": "filename or null", "error": "error message or null" }
```

Store all results in a JavaScript array for use in Phase 7.

### Phase 6: Full QA with Playwright

Broader quality checks across all pages.

#### QA Check A: Page Load Verification
For each HTML page in the project:
1. Navigate to the page using `browser_navigate` with `file://` URL
2. Use `browser_snapshot` to verify the page loaded (has content)
3. Use `browser_console_messages` with level `error` to check for console errors
4. Record result as `{ "name": "Page load: [filename]", "category": "page-load", "passed": true/false, "error": "..." }`

#### QA Check B: Navigation Links
Test navigation paths between pages:
1. Find links on each page using `browser_snapshot`
2. Click links and verify navigation using `browser_evaluate` with `window.location.href`
3. Record each as `{ "name": "Nav: [description]", "category": "navigation", "passed": true/false, "error": "..." }`

#### QA Check C: Responsive Layout
1. Use `browser_resize` to set viewport to `width: 768, height: 1024`
2. Navigate to key pages
3. Use `browser_evaluate` to check no horizontal overflow: `document.documentElement.scrollWidth <= document.documentElement.clientWidth`
4. Take screenshot at mobile width
5. Reset to `width: 1280, height: 720`
Record as `{ "name": "Responsive: [page] at 768px", "category": "visual", "passed": true/false, "screenshot": "..." }`

#### QA Check D: Console Error Audit
1. For each page, navigate and collect `browser_console_messages` at `error` level
2. Record any errors found
Record as `{ "name": "Console audit: [page]", "category": "console", "passed": true/false, "error": "list of errors" }`

#### QA Check E: Edge Cases
1. **XSS in input fields:** Type `<script>alert('xss')</script>` in text inputs. Verify no dialog appears.
2. **Very long input:** Type a 500-character string in text inputs. Verify no page crash.
3. **Rapid double-click submit:** Fill valid form data, click submit twice rapidly. Verify graceful handling.
Record each as `{ "name": "Edge case: [description]", "category": "edge-case", "passed": true/false, "error": "..." }`

### Phase 7: QA Report as Interactive HTML

Generate `qa-report.html` with the combined results from Phase 5 and Phase 6, using the Anthropic web designer aesthetic that matches the code review report.

The HTML structure must include:
- `.header` with indigo gradient, title, subtitle, and `[TIMESTAMP]` build stamp
- `.meta-bar` below the header: a horizontal bar (dark background, e.g. `var(--gray-800)`) displaying key metadata in a flex row with pill-style items: **Repository** (org/repo name from `git remote`), **Branch** (current branch name), **Generated** (date and time, e.g. "2 Mar 2026, 23:00"), **Commit** (short SHA of HEAD). Each item has a small uppercase label and value. Use `var(--font-mono)` for values.
- `.summary` grid with 4 `.summary-card` elements (total/passed/failed/pass rate, computed dynamically)
- `.progress-panel` with overall progress bar (green/amber/red gradient based on rate)
- `.breakdown-panel` with category rows: icon, name, mini progress bar, and score
- `.category-section` cards for each test category with `.category-head` (title + badge) and `.category-tests` list
- Each test row shows a circular pass/fail status indicator, test name, error details (if failed), screenshot links
- Failed tests get checkboxes (pre-checked) for user selection
- `.actions-panel` with "Generate Fix Instructions" button, readonly textarea, "Copy to Clipboard" button with 3-tier clipboard fallback, and `.copy-notice` div
- JavaScript: `render()`, `generateFixInstructions()`, `copyToClipboard()`, `fallbackCopy()`, `showCopyNotice()`
- Category order: integration, page-load, navigation, visual, console, edge-case

Data placeholder: `var results = [QA_RESULTS_JSON];`

If `qa-report.html` already exists in the project root, use it as the canonical reference implementation — replicate its exact CSS and HTML structure, replacing only the data (`[QA_RESULTS_JSON]`) and build stamp (`[TIMESTAMP]`). Otherwise, generate the HTML from the description above.

**Important:** Replace placeholders:
- `[QA_RESULTS_JSON]` - JSON array of test results from Phases 5 and 6
- `[TIMESTAMP]` - Current date/time string so users can verify they see the latest version
- `[REPO_NAME]` - Repository name from `git remote get-url origin` (e.g., `balgaly/guessit`)
- `[BRANCH_NAME]` - Current branch from `git branch --show-current`
- `[GENERATED_DATE]` - Human-readable date/time (e.g., `2 Mar 2026, 23:00`)
- `[COMMIT_SHA]` - Short SHA of HEAD from `git rev-parse --short HEAD`

7. **Always open the QA report** in the browser automatically — even when all tests pass. The user wants to see the full results visually.
8. **Print a summary** to the terminal: total tests, passed, failed, pass rate, and category breakdown.
9. **Wait for user input** - tell user to:
   - Review the QA report in the browser
   - If all passed: click "Generate Fix Instructions" to get the `SHIP_WORKFLOW_QA_PASS` token
   - If failures exist: select which to fix, click "Generate Fix Instructions"
   - Copy the generated text
   - Paste it back to Claude Code

### Phase 8: QA Fix Loop and Workflow Tokens

When user pastes back instructions:

- **`SHIP_WORKFLOW_QA_PASS:`** - All QA passed (or user chose to skip failures). Proceed to Final Commit (Phase 9).
- **`SHIP_WORKFLOW_QA_FIX:`** - User selected failures to fix. Parse the failures, fix the underlying code issues, then re-run Phases 5-6 and regenerate the QA report. Repeat until the user sends `SHIP_WORKFLOW_QA_PASS`.

### Phase 9: Final Commit

1. **Stage all changes:**
   ```bash
   git add .
   ```

2. **Create descriptive commit message** following conventional commits:
   ```
   <type>: <short description>

   <detailed description of changes>

   - Bullet points of key changes
   - Fixed issues from code review
   - Test results: X passing unit tests
   - QA results: X/Y integration and QA tests passed

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
   ```

3. **Commit:**
   ```bash
   git commit -m "$(cat <<'EOF'
   [commit message here]
   EOF
   )"
   ```

4. **Show commit summary** to user:
   - Commit hash
   - Files changed
   - Lines added/removed
   - Unit test results
   - QA test results
   - Final status

### Phase 10: Cleanup, Merge to Main & Branch Deletion

1. **Clean up generated artifacts** before merging:
   - Delete `code-review-report.html` if it exists
   - Delete `qa-report.html` if it exists
   - Delete any Playwright screenshots taken during QA (e.g., `*.png` files generated in the project root by Playwright)
   - Stage the deletions:
     ```bash
     git rm -f code-review-report.html qa-report.html 2>/dev/null; git rm -f .playwright-mcp/*.png 2>/dev/null; git status
     ```
   - If there are deletions to commit, create a cleanup commit:
     ```bash
     git commit -m "$(cat <<'EOF'
     chore: clean up generated review and QA artifacts

     Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
     EOF
     )"
     ```

2. **Merge to main:**
   - Get the current branch name:
     ```bash
     git branch --show-current
     ```
   - Detect the default branch (main or master):
     ```bash
     git branch --list main master | head -1 | tr -d ' *'
     ```
   - Switch to default branch and merge:
     ```bash
     git checkout <default-branch> && git merge --no-ff <branch-name> -m "$(cat <<'EOF'
     Merge branch '<branch-name>'

     Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
     EOF
     )"
     ```
   - If merge conflicts occur, inform the user and ask for guidance. Do not auto-resolve merge conflicts.

3. **Delete the feature branch:**
   ```bash
   git branch -d <branch-name>
   ```

4. **Show final status** to user:
   - Confirm merge was successful
   - Confirm branch was deleted
   - Show `git log --oneline -5` to display the merge commit

## Commit Message Types

Use appropriate type:
- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code refactoring
- `test`: Adding tests
- `docs`: Documentation
- `style`: Code style changes
- `perf`: Performance improvements
- `chore`: Maintenance tasks

## Success Criteria

Ship is complete when:
- All selected code review issues fixed
- All unit tests passing
- All QA tests passing (or user explicitly skipped failures)
- Changes committed with descriptive message including QA results
- Generated artifacts (HTML reports, screenshots) cleaned up
- Branch merged to default branch
- Feature branch deleted
- User informed of final status

## Error Handling

If something goes wrong:
- Explain what failed
- Ask user for guidance
- Do not proceed without confirmation

### Playwright-Specific Error Handling

- **Playwright MCP not available:** If `browser_navigate` tool is not available or returns an error indicating the MCP server is not connected, skip Phases 5-7 entirely. Print a warning: "Playwright MCP tools not available. Skipping integration testing and QA phases. Proceeding to final commit." Then jump to Phase 9.
- **Browser not installed:** If Playwright returns an error about missing browser/Chromium, run `browser_install` first, then retry the navigation.
- **File URL format:** Use `file:///C:/path/to/file.html` format on Windows, `file:///path/to/file.html` on Mac/Linux. If navigation fails, try alternate slash format. Use the project's absolute path from the working directory.
- **Test hangs or timeout:** If a Playwright operation appears stuck, use `browser_close` to close the browser and log the test as failed with error "Test timed out".
- **Page errors:** If `browser_console_messages` returns errors during a test, capture them in the test result's error field but don't necessarily fail the test unless the test specifically checks for console errors.

## User Communication

- Be concise during the workflow
- Only report issues if you need user input
- Silent testing and fixing is preferred
- Final summary should be comprehensive
