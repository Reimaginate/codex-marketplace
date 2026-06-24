---
name: design-hub
description: Use Design Hub MCP tools to read, search, create, or update Reimaginate Design Hub workspaces and pages, including specs, implementation plans, requirements, architecture notes, decisions, design documentation, Orchestrator workflow DSL blocks, and sequence diagram DSL blocks. Use when Codex needs Design Hub workspace context or needs to author Design Hub page content with `diagram.workflow.v1.1.0` or `diagram.sequence.v1.1.0`. Do not use for public docs, generic coding questions, or content already supplied in the conversation.
---

# Design Hub

Use the Design Hub MCP tools when the user asks about Design Hub workspaces, views, pages, specs, plans, requirements, architecture notes, implementation documentation, decisions, design notes, or Design Hub page content that may live in Design Hub.

Do not use Design Hub tools for general public documentation, generic coding questions, information already supplied in the current conversation, unrelated repository inspection, or speculative answers that do not require Design Hub context.

## Access Workflow

### CLI vs MCP tools

Use the structured Design Hub MCP tools for workspace and page work. Do not shell out to the `designhub` CLI for operations that have MCP tools available.

If the Design Hub MCP tools or namespace are not exposed in the current Codex turn, stop and report that the Design Hub MCP server is not available in the thread. Do not use the local `designhub` CLI as a fallback for workspace, view, TOC, page, draft, search, asset, auth, or profile operations.

The CLI is still used by humans to configure authentication and by Codex to start the MCP transport:

- Human Codex plugin install: `designhub mcp install codex`.
- Human setup: `designhub login`. The default profile targets `https://api.designhub.online`; use `designhub setup --profile <name> --api-url <api-url>` only for additional environments.
- Human draft commands: `designhub drafts get`, `designhub drafts save`, `designhub drafts publish`, and `designhub drafts discard`.
- MCP transport: `designhub mcp --stdio`.

The hosted plugin configuration already uses this transport shape in `.mcp.json`.

### Profile safety

Codex must not manipulate Design Hub CLI or shared profile selection in any way.

Allowed:

- Use `auth_status` to inspect whether the current configured profile is usable.
- Use `auth_refresh` only when the already-selected profile has existing local auth that needs refreshing.
- Tell the user to run `designhub login` when authentication is missing.

Forbidden:

- Do not create, update, switch, rename, remove, or select Design Hub profiles.
- Do not change profile targets or the current profile.
- Do not pass `--profile` to `designhub` commands.
- Do not run `designhub setup`, `designhub logout`, `profiles use`, `profiles set`, `profiles remove`, `profiles rename`, `profiles targets set`, `profiles targets unset`, or equivalent profile mutation commands.
- Do not edit local profile or credential files directly.

If a task fails because the wrong profile is selected, a profile target is missing, or the profile configuration is invalid, stop and report the exact issue. Ask the user to fix profile selection or profile configuration outside Codex, then retry after they confirm it is fixed.

### Tool workflow

1. Start with `auth_status` before a task that needs Design Hub access.
2. If authentication is missing, tell the user to run:

```bash
designhub login
```

3. Use `auth_refresh` only when existing local auth needs refreshing.
4. Use `list_workspaces` and `list_views` before assuming workspace or view ids.
5. Use `get_toc`, `list_pages`, or `search_pages` to find relevant pages.
6. Use `get_page` before updating an existing page so you have the current content and `versionId`.
7. Keep results scoped. Use limits and follow-up searches instead of pulling broad content.

Common MCP tools:

- Workspace discovery: `list_workspaces`, `list_views`, `get_toc`.
- Page discovery and content: `list_pages`, `get_page`, `search_pages`.
- Page writes: `create_page`, `update_page`.
- Draft writes: `get_active_draft`, `save_draft`, `publish_draft`, `discard_draft`.
- TOC item writes: `insert_toc_item`, `remove_toc_item`.
- Workspace assets: `list_workspace_assets`, `download_workspace_asset`, `upload_workspace_asset`.

## TOC Groups

In Design Hub workspace navigation, a "group" means a table-of-contents group item.

When the user asks to create a group:

- Create a TOC item with `insert_toc_item`; do not create a page, workspace, folder, diagram group, security group, or data model group.
- Use a TOC item shaped as `type:"group"` and `label:<requested group name>`.
- Do not include `linkedSectionId` for a normal group unless the user explicitly asks to link or mirror TOC sections.
- Insert only one TOC item at a time. Do not submit a group with populated `children`; insert child pages or child groups separately.
- Use `parentId` only when the user asks for a nested group under an existing TOC group. Omit `parentId` for a root-level group.
- The MCP adapter clears any supplied TOC item id before insertion, and the server generates the inserted TOC item id. After insertion, use the returned TOC contents to find the new group id before adding children.

When the user asks to create pages under a group:

- First call `get_toc` and find the named group. If it does not exist and the user's wording implies it should be created, insert the TOC group first.
- Create each requested page with `create_page`, then insert each page into the TOC with `insert_toc_item` using the group's TOC item id as `parentId`.
- Use a page TOC item shaped as `type:"page"`, `label:<created page title>`, and `linkedSectionId:<created page id>`.
- Refresh or reuse the latest TOC `revision` returned after each TOC write as the next `expectedRevision`.
- Do not place those page items at the root unless the user explicitly asks for root placement.

## Write Boundary

The Design Hub integration can read content freely and can write through tools that require explicit confirmation.

Allowed:

- Check auth status.
- Refresh local auth.
- List workspaces.
- List views.
- Read TOCs.
- List pages.
- Get page content.
- Search pages.
- Create pages with `create_page` and `confirm:true`.
- Update pages with `update_page` and `confirm:true`.
- Get active drafts with `get_active_draft`.
- Save existing page drafts with `save_draft` and `confirm:true`.
- Publish saved drafts with `publish_draft` and `confirm:true`.
- Discard saved drafts with `discard_draft` and `confirm:true`.
- Insert TOC items with `insert_toc_item`, `confirm:true`, and the current `expectedRevision` from `get_toc`.
- Remove TOC items with `remove_toc_item`, `confirm:true`, and the current `expectedRevision` from `get_toc`.
- List, download, or upload workspace assets with `list_workspace_assets`, `download_workspace_asset`, and `upload_workspace_asset`; upload requires `confirm:true`.

For draft-only page work:

- Use `get_active_draft`, `save_draft`, `publish_draft`, and `discard_draft`.
- Do not call `create_page` or `update_page` when the user asks to create or save a draft without publishing.
- Use `save_draft` only for existing pages. If the target page does not exist, report that draft-only creation of brand-new pages is not supported.
- Do not add draft-only saves to the TOC. TOC insertion applies to published `create_page` only.
- For a first draft save, use the active draft's `baseVersionId`. For an existing saved draft, use its `id` as `draftId` and its `draftRevision` as `expectedDraftRevision`.

For page creation:

- After `create_page` succeeds, add the new page to the root of the default workspace TOC unless the user explicitly requests a different placement or says not to add it to the TOC.
- Use the created page response `id` and `title`.
- Call `get_toc` for the workspace without `viewId`.
- Call `insert_toc_item` with `confirm:true`, `expectedRevision` from `get_toc`, `position:"append"`, and no `parentId`.
- Use a TOC item shaped as `type:"page"`, `label:<created page title>`, and `linkedSectionId:<created page id>`.
- If the TOC insert fails after page creation, report that the page was created but not placed in the TOC. Do not delete the page as cleanup.

For page updates:

- Call `get_page` first.
- Pass the returned `versionId` as `expectedVersionId`.
- Use `allowStaleVersion:true` only when the user explicitly accepts a stale overwrite.
- Do not discard drafts unless the user explicitly accepts that risk and the tool supports it.

Do not change workspace settings, view settings, permissions, production configuration, or unrelated pages unless explicitly requested.

## Design Hub DSL Blocks

Design Hub pages use fenced Markdown code blocks for diagram DSL. Prefer the current Design Hub DSLs over Mermaid when they fit the request.

### Orchestrator Workflows

Use `diagram.workflow.v1.1.0` for executable Orchestrator workflow YAML. This is not the old actor/activity/flow diagram schema.

Canonical fence:

````markdown
```diagram.workflow.v1.1.0
document:
  dsl: "1.0.0"
  namespace: default
  name: sample-workflow
  version: "0.0.1"
  extensions:
    eventTriggers: []
do:
  - Start:
      call: ReverseText
      with:
        Text: hello
      then: Finish
  - Finish:
      terminate:
        status: Completed
```
````

Workflow rules:

- A workflow must define `do` at the root or under `document.do`.
- Each `do` item must contain exactly one named task definition.
- Supported task kinds include `call`, `run`, `map`, `listen`, `wait`, `switch`, `do`, `fork`, `try`, `terminate`, `raise`, `emit`, and `stash`.
- Root task order does not imply routing. Use explicit `then` targets or branch targets.
- Event-driven workflows declare triggers under `document.extensions.eventTriggers`.
- Do not use Mermaid flowchart syntax inside `diagram.workflow.v1.1.0`.
- Do not use the legacy `actors`, `artefacts`, `activities`, `events`, and `flows` schema for `diagram.workflow.v1.1.0`.

For detailed workflow grammar, use these repository guides when they are available in the current workspace:

- `docs/orchestrator-yaml-dsl-guide.md`
- `docs/orchestrator-dsl-specification.md`
- `docs/orchestrator-expression-user-guide.md`

### Sequence Diagrams

Use `diagram.sequence.v1.1.0` for chronological participant interactions, handoffs, and process steps over time.

Canonical fence:

````markdown
```diagram.sequence.v1.1.0
title: Disposal review process
participants:
  - id: purview
    label: Purview
  - id: crm
    label: WBS CRM
timeline:
  - flow:
      from: purview
      to: crm
      label: Retention expires
      kind: sync
  - note:
      of:
        - crm
      text: Create a review case for the responsible owner
```
````

Sequence rules:

- `participants` is required and ordered left to right.
- Participant entries use `id` and `label`.
- `timeline` items use canonical YAML objects: `flow`, `note`, or `group`.
- `flow.kind` may be `sync`, `async`, or `reply`.
- `group` supports combined fragments such as `alt`, `opt`, `loop`, `par`, `break`, `critical`, and `group`.
- Do not use Mermaid arrow lines such as `a -> b: message` inside this DSL.

For detailed sequence syntax, use `docs/sequence-diagram-user-guide.md` when it is available in the current workspace.
