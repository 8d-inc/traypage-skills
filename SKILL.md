---
name: traypage
description: Use when a user wants an AI coding agent to publish HTML or Markdown to TrayPage, share review/share URLs, inspect review comments, and publish revised versions. Prefer TrayPage MCP tools when available; this skill focuses on the TrayPage publish-review-revise workflow rather than app development.
---

# TrayPage

Use this skill when the user wants to publish a page to TrayPage, share it for review,
inspect review comments, or publish a revised version. TrayPage is a page publishing and
review workflow for AI-generated HTML and Markdown.

## Tool Preference

Prefer TrayPage MCP tools when they are available in the agent:

- `publish_page`
- `publish_page_version`
- `get_page_comments`
- `get_page_revision_prompt`
- `list_pages`

If the MCP tools are not available, tell the user to connect TrayPage at user/global scope with
`https://tray.page/api/mcp`. Do not spend time hand-rolling HTTP calls unless the user explicitly
asks for automation outside an MCP-capable client.

TrayPage may add a CLI later. When a `traypage` CLI is available, use MCP for interactive agent
work and use the CLI for shell scripts, CI, and MCP-unavailable environments.

## Publish a New Page

1. Choose `content_type`.
   - Use `text/markdown` for prose-heavy reports, plans, tables, and documentation.
   - Use `text/html` for designed pages, dashboards, interactive artifacts, and rich layout.
2. For HTML, produce one self-contained document. Inline the data the page needs at publish time.
3. Call `publish_page` with:
   - `page_title`
   - `content`
   - `content_type`
   - optional `changelog`
   - optional `organization_slug` and `project_slug` only when the user asks for a non-default target.
4. Return the important result fields:
   - `review_url` for authenticated review, comments, and version comparison.
   - `share_url` for human sharing through TrayPage's share gate.
   - `page_id` for later comment fetching and revisions.

Do not give users `/viewer/...` URLs directly. Viewer URLs are implementation details.

## Targeting Organization and Project

By default, publish without `organization_slug` or `project_slug`. TrayPage uses the default
project selected during OAuth consent, or the project attached to the API token.

Specify targets only when needed:

- Use `project_slug` when the user names another authorized project in the same organization.
- Use both `organization_slug` and `project_slug` when the user names another organization.
- If the target is unclear, ask one short question rather than guessing.

If TrayPage returns `Organization not found`, `Project not found`, or `project_restricted`, explain
that the MCP authorization or API token does not cover that target.

## Review and Revise

When the user asks to apply TrayPage comments:

1. Identify the page.
   - Prefer a saved `page_id` from the prior publish result.
   - If missing, call `list_pages` and match by title, slug, folder, or project.
2. Call `get_page_revision_prompt` with `page_id`.
   - Pass `version_number` only when the user asks to revise a specific version.
3. Use the revision prompt as the concrete change list.
   - If it is empty or ambiguous, call `get_page_comments` with `status: "open"` and summarize
     the actionable comments before editing.
4. Revise the original content. Preserve the content type unless the user asks to change it.
5. Call `publish_page_version` with:
   - `page_id`
   - updated `content`
   - `content_type`
   - concise `changelog`
6. Return the new `version_number` and `share_url`, and tell the user reviewers can continue from
   the existing review page.

Do not claim comments are resolved unless TrayPage returns that state or the user resolves them in
the UI. Publishing a new version addresses feedback; it does not automatically close review threads.

## Inspect Comments

Use `get_page_comments` when the user asks what reviewers said, wants a summary before editing, or
needs to audit open versus resolved feedback.

Recommended arguments:

- `status: "open"` for normal revision work.
- `status: "resolved"` for audit or history requests.
- `version_number` when comparing feedback on a specific version.

Report comments as actionable work items with enough context to revise accurately. Avoid dumping
long comment histories unless the user asks.

## HTML Page Rules

TrayPage HTML pages run in a strict sandbox. Follow these rules when generating or debugging HTML:

- Make the page a single self-contained HTML document.
- Inline all data needed for rendering; runtime `fetch`, XHR, WebSocket, SSE, and `sendBeacon` are blocked.
- Do not use `localStorage`, `sessionStorage`, IndexedDB, or cookies.
- Handle forms in JavaScript with `preventDefault()`; native form submission is blocked.
- Use pinned `https:` CDN URLs for libraries when needed.
- Replace iframes and live embeds with normal links.
- Do not interpolate private data into image, script, style, or font URLs.

Markdown pages are converted to HTML by TrayPage and usually do not need these sandbox-specific
rules.
