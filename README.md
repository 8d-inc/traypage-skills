# TrayPage skill and Claude Code plugin

Create HTML or Markdown pages, collect browser review feedback, and publish revised
versions with [TrayPage](https://tray.page).

The Claude Code plugin bundles the existing TrayPage skill and the HTTP MCP
connection. Both installation methods use the same root `SKILL.md`; there is no
second copy to keep in sync.

## Claude Code plugin

Run these commands in Claude Code:

```text
/plugin marketplace add 8d-inc/traypage-skills
/plugin install traypage@traypage
```

Reload plugins with `/reload-plugins`, or start a new session. Open `/mcp`, select
the TrayPage server added by the plugin, and authenticate in the browser. Choose
the organization, accessible projects, and default project on the consent screen.
The plugin uses OAuth; it does not contain or ask you to paste an API token.

Then ask naturally, or invoke the skill explicitly:

```text
/traypage:page Create a Markdown report as a draft and give me its review URL.
```

If you already installed the standalone skill, it can remain installed; the plugin
skill is exposed as `traypage:page` by the plugin; the standalone skill keeps its
`traypage` name. The command prefix identifies the plugin. Existing MCP configurations for the same
endpoint may take precedence over the plugin connection. Use the server shown in
`/mcp` and authenticate it; you do not need to add a second connection manually.

Update an installed plugin with:

```bash
claude plugin marketplace update traypage
claude plugin update traypage@traypage
```

Reload plugins or start a new session after updating.

## Standalone skill for Claude Code, Codex, and other agents

The existing installation continues to work:

```bash
npx skills add 8d-inc/traypage-skills -g
```

This installs the skill only. Connect MCP separately using the
[MCP setup guide](https://tray.page/docs/en/mcp).

## Connected browser review

MCP creates drafts, publishes versions, and handles review comments. To let an
agent wait for browser review completion and continue with revisions, install the
CLI with Node.js 22 or newer and authenticate it separately:

```bash
npm install -g @traypage/cli
traypage auth login
traypage auth status
```

The MCP login and CLI login are separate. Select matching organization/project
access when using them together. After creating a draft, the agent can run:

```bash
traypage review start --page <page_id> --version <version_number> --watch
```

Open the session-specific review URL printed by that command. When you finish the
review in the browser, the waiting agent receives the result and can create a
revised draft. Starting a review does not publish the draft. See the
[CLI guide](https://tray.page/docs/en/cli) for the full workflow.

## Local development

Load this checkout for a single session without installing it globally:

```bash
claude --plugin-dir /absolute/path/to/traypage-skills
```

To exercise the marketplace install path before merging, use the local checkout:

```text
/plugin marketplace add /absolute/path/to/traypage-skills
/plugin install traypage@traypage
```

If the `traypage` marketplace already points to GitHub, use `--plugin-dir` for local
testing rather than replacing your existing marketplace.

The default MCP endpoint is `https://tray.page/api/mcp`. Override it for a local or
staging TrayPage instance before launching Claude Code:

```bash
TRAYPAGE_MCP_URL=http://localhost:3000/api/mcp \
  claude --plugin-dir /absolute/path/to/traypage-skills
```

Authenticate against that instance via `/mcp`. If you also use connected review
sessions there, set the CLI API base URL to the origin, without `/api/mcp`:

```bash
TRAYPAGE_API_BASE_URL=http://localhost:3000 traypage auth login
TRAYPAGE_API_BASE_URL=http://localhost:3000 \
  traypage review start --page <page_id> --version <version_number> --watch
```

Validate the package using Claude Code:

```bash
claude plugin validate --strict .claude-plugin/plugin.json
claude plugin validate --strict .claude-plugin/marketplace.json
```

Package contents:

```text
.claude-plugin/plugin.json       plugin metadata; exposes the root skill as page
.claude-plugin/marketplace.json  marketplace catalog
.mcp.json                       HTTP MCP connection with OAuth
SKILL.md                        shared create/review/publish/revise instructions
agents/openai.yaml              existing Codex skill metadata
```

The plugin does not require a build, bundled server, or startup hook. Page storage,
permissions, comments, and version management run in TrayPage itself.
