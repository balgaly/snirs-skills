# Ship Workflow Guide

## Overview

The `/ship` command automates your complete code shipping workflow:

1. Initial commit
2. Code review
3. Interactive HTML report
4. Iterative fixes
5. Unit test execution
6. Integration testing with Playwright
7. Full QA with Playwright
8. QA report as interactive HTML
9. Final commit
10. Cleanup, merge to main & branch deletion

## How to Use

### Basic Usage

```
/ship
```

Or with a custom commit message:

```
/ship Add user authentication feature
```

## The Workflow

### Step 1: Initial Commit & Review

Claude will:
- Commit your current changes as "WIP: Changes ready for review"
- Run comprehensive code review
- Generate an interactive HTML report

### Step 2: Review & Select Fixes

An HTML file (`code-review-report.html`) will open in your browser showing:

- **Critical Issues** (pre-checked)
- **Warnings**
- **Suggestions**

**Your actions:**
1. Review each issue
2. Check/uncheck items you want fixed
3. Click "Generate Fix Instructions"
4. Copy the generated text (works on `file://` URLs too)

### Step 3: Apply Fixes

**Paste the copied text back to Claude Code**

Claude will:
- Parse your selected fixes
- Apply each fix automatically
- Rerun code review
- If new issues found, repeat Step 2

### Step 4: Unit Testing (Automatic)

Claude will:
- Detect and run test files automatically
- Fix any test failures
- Rerun until all pass
- Only notify you if it cannot fix issues

### Step 5: Integration Testing with Playwright

Claude will use Playwright MCP tools to test the application end-to-end in a real browser:
- Interactive elements (forms, buttons, modals)
- Input validation and error states
- Dynamic UI behavior
- DOM state verification via `browser_evaluate`

### Step 6: Full QA with Playwright

Broader quality checks across all pages:
- **Page loads:** All pages load without console errors
- **Navigation:** All links work correctly between pages
- **Responsive:** No horizontal overflow at 768px width
- **Console audit:** No error-level messages on any page
- **Edge cases:** XSS in inputs, very long input, rapid double-click submit

### Step 7: QA Report

An interactive HTML file (`qa-report.html`) opens in your browser showing:
- Stats dashboard (total, passed, failed, pass rate)
- Test results grouped by category
- Pass/fail icons with error details and screenshot links
- Checkboxes on failed tests for selection

**Your actions:**
1. Review the QA results
2. Select which failures to fix (failed tests are pre-checked)
3. Click "Generate Fix Instructions"
4. Copy and paste back to Claude Code

If all tests passed, generate instructions will produce `SHIP_WORKFLOW_QA_PASS` and you can proceed directly.

### Step 8: QA Fix Loop

If you selected failures to fix:
- Claude fixes the underlying code issues
- Reruns integration tests and QA
- Generates a new QA report
- Repeats until all pass or you choose to proceed

### Step 9: Final Commit

Claude will:
- Create a descriptive commit message following conventional commits
- Include unit test and QA results in the commit message
- Commit all changes
- Show you the summary

### Step 10: Cleanup, Merge & Branch Deletion

Claude will:
- Delete generated artifacts (`code-review-report.html`, `qa-report.html`, Playwright screenshots)
- Commit the cleanup
- Merge the feature branch into default branch (using `--no-ff` for a merge commit)
- Delete the feature branch
- Show the final merge status

If merge conflicts occur, Claude will ask for your guidance.

## Requirements

- **Playwright MCP server** must be configured for integration testing and QA phases (Steps 5-7). If not available, these phases are skipped with a warning and the workflow proceeds to final commit.
- Node.js for running unit tests.

## Special Commands

If you want to skip all code review fixes and proceed:

In the HTML report, uncheck all items and generate instructions. The output will be:
```
SHIP_WORKFLOW_CONTINUE: All review items cleared. Proceed to testing and final commit.
```

Paste this back to Claude to skip fixes and go straight to testing.

For the QA report, if all tests pass or you want to skip failures:
```
SHIP_WORKFLOW_QA_PASS: All QA tests passed or no failures selected. Proceed to final commit.
```

## Commit Message Format

Final commits follow conventional commits:

```
<type>: <short description>

<detailed description>

- Key change 1
- Key change 2
- Fixed issues from code review
- Test results: X passing unit tests
- QA results: X/Y integration and QA tests passed

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `style`, `perf`, `chore`

## Troubleshooting

**HTML report doesn't open?**
- Check `code-review-report.html` in your project root
- Open it manually in your browser

**Copy to clipboard not working?**
- The button uses a 3-tier fallback: Clipboard API -> execCommand -> manual select
- On `file://` URLs, the fallback tiers handle the copy
- If all else fails, the text is auto-selected for manual Ctrl+C

**Tests keep failing?**
- Claude will attempt to fix and rerun
- If it can't fix, it will ask for your help

**Playwright MCP not available?**
- Ensure your `.mcp.json` or MCP settings include the Playwright server
- Integration testing and QA phases will be skipped if Playwright is not configured
- Unit tests and code review still run normally

**Playwright browser not installed?**
- Claude will attempt to run `browser_install` automatically
- If that fails, check your Playwright MCP server configuration
