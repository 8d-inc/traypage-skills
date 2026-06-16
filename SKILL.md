---
name: traypage
description: Use when a user wants an AI coding agent to create HTML or Markdown pages in TrayPage, review and share them, publish a reviewed version, inspect review comments, or create revised draft versions. Prefer TrayPage MCP tools when available; this skill focuses on the TrayPage create-review-publish-revise workflow rather than app development.
---

# TrayPage

Use this skill when the user wants to create a page in TrayPage, share it for review,
publish a reviewed version, inspect review comments, or create a revised draft version.
TrayPage is a page publishing and review workflow for AI-generated HTML and Markdown.

## Core Principles

Follow these principles for all TrayPage work:

1. **Workflow first**: focus on the create -> review -> publish -> revise loop, not on
   configuring tools unless the user is blocked on connection.
2. **Prefer the official TrayPage tool surface**: use TrayPage MCP tools when they are available.
   Do not invent direct API calls or URL shapes when a supported tool can do the job.
3. **Default target unless told otherwise**: create pages in the authorized default project unless
   the user names an organization or project.
4. **Human URLs only**: return `review_url` and `share_url`; never hand users `/viewer/...` URLs.
5. **Keep creation and publishing separate**: `create_page` and `create_page_version` create draft
   versions. Call `publish_version` only after the user wants a version shown from the share URL.
6. **Do not overclaim review state**: creating or publishing a revised version does not
   automatically resolve comments.
7. **Check current docs when exact setup or behavior matters**: TrayPage can change. If the task
   depends on setup details, tool arguments, scopes, or sandbox behavior, consult the public docs:
   `https://tray.page/docs/en/mcp`, `https://tray.page/docs/en/quickstart`, and
   `https://tray.page/docs/en/page-capabilities`.

## Access Methods

Prefer TrayPage MCP tools when they are available in the agent:

- `create_page`
- `create_page_version`
- `publish_version`
- `set_page_visibility`
- `get_page_comments`
- `get_page_revision_prompt`
- `list_pages`

If the MCP tools are not available, tell the user to connect TrayPage at user/global scope with
`https://tray.page/api/mcp`. Do not spend time hand-rolling HTTP calls unless the user explicitly
asks for automation outside an MCP-capable client.

Do not assume a `traypage` CLI exists. If the current TrayPage docs or the local environment show
an official CLI, prefer MCP for interactive agent work and use the CLI for shell scripts, CI, and
MCP-unavailable environments.

## Create a New Draft Page

1. Choose `content_type`.
   - Use `text/markdown` for prose-heavy reports, plans, tables, and documentation.
   - Use `text/html` for designed pages, dashboards, interactive artifacts, and rich layout.
2. For HTML, produce one self-contained document. Inline the data the page needs at creation time.
3. Call `create_page` with:
   - `page_title`
   - `content`
   - `content_type`
   - optional `changelog`
   - optional `organization_slug` and `project_slug` only when the user asks for a non-default target.
4. Return the important result fields:
   - `review_url` for authenticated review, comments, and version comparison.
   - `share_url` for human sharing through TrayPage's share gate after a version is published.
   - `page_id` for later comment fetching and revisions.
   - `version_number` for publishing the reviewed version.
5. Tell the user the created version is a draft. The share URL is stable, but it will not show the
   draft body until a version is published.

Do not give users `/viewer/...` URLs directly. Viewer URLs are implementation details.

## Publish a Reviewed Version

Use `publish_version` when the user says the version is ready, wants the share link to show it, or
asks to publish a specific version.

Call `publish_version` with:

- `page_id`
- `version_number`

Publishing makes that version live on the stable `share_url`. Only one version can be live at a
time; publishing a new version returns the previous live version to draft. This does not change who
can open the share URL. Use `set_page_visibility` when the user asks to change the audience.

## Targeting Organization and Project

By default, create without `organization_slug` or `project_slug`. TrayPage uses the default
project selected during OAuth consent, or the project attached to the API token.

Specify targets only when needed:

- Use `project_slug` when the user names another authorized project in the same organization.
- Use both `organization_slug` and `project_slug` when the user names another organization.
- If the target is unclear, ask one short question rather than guessing.

If TrayPage returns `Organization not found`, `Project not found`, or `project_restricted`, explain
that the MCP authorization or API token does not cover that target.

## Auth and Scope Failures

For authentication or authorization failures, report the failing action and the concrete error.
Then guide the user to re-authenticate the TrayPage connection or use an API token with the needed
scope:

- `page:write` for `create_page`, `create_page_version`, `publish_version`, and `set_page_visibility`.
- `comment:read` for `get_page_comments`.
- `revision_prompt:read` for `get_page_revision_prompt`.
- `project:read` or `folder:read` when listing projects, folders, or page targets.

## Review and Revise

When the user asks to apply TrayPage comments:

1. Identify the page.
   - Prefer a saved `page_id` from the prior create result.
   - If missing, call `list_pages` and match by title, slug, folder, or project.
2. Call `get_page_revision_prompt` with `page_id`.
   - Pass `version_number` only when the user asks to revise a specific version.
3. Use the revision prompt as the concrete change list.
   - If it is empty or ambiguous, call `get_page_comments` with `status: "open"` and summarize
     the actionable comments before editing.
4. Revise the original content. Preserve the content type unless the user asks to change it.
5. Call `create_page_version` with:
   - `page_id`
   - updated `content`
   - `content_type`
   - concise `changelog`
6. Return the new `version_number`, `review_url`/`share_url` when available, and tell the user the
   revised version is a draft until published.
7. If the user wants the revised version to appear from the share URL, call `publish_version` with
   the new `version_number` after review.

Do not claim comments are resolved unless TrayPage returns that state or the user resolves them in
the UI. Creating or publishing a new version addresses feedback; it does not automatically close
review threads.

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

## Output Expectations

After creating, publishing, or revising, give the user:

- what was created, published, or changed,
- `review_url` when available,
- `share_url` when available,
- `page_id` if it will help with future revisions,
- the organization/project target only if it was explicit or relevant.

If this skill gives wrong or outdated guidance, or the user says the TrayPage workflow should work
differently, offer to file feedback against `https://github.com/8d-inc/traypage-skills`.
