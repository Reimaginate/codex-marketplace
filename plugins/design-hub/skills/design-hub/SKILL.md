---
name: design-hub
description: Use Design Hub MCP tools to read, search, create, or update Reimaginate Design Hub workspaces and pages, including specs, implementation plans, requirements, architecture notes, decisions, design documentation, Orchestrator workflow DSL blocks, sequence diagram DSL blocks, and entity relationship diagram DSL blocks. Use when Codex needs Design Hub workspace context or needs to author Design Hub page content with `diagram.workflow.v1.1.0`, `diagram.sequence.v1.1.0`, or `yaml:erd`. Do not use for public docs, generic coding questions, or content already supplied in the conversation.
---

# Design Hub

Use the Design Hub MCP tools when the user asks about Design Hub workspaces, views, pages, specs, plans, requirements, architecture notes, implementation documentation, decisions, design notes, or Design Hub page content that may live in Design Hub.

Do not use Design Hub tools for general public documentation, generic coding questions, information already supplied in the current conversation, unrelated repository inspection, or speculative answers that do not require Design Hub context.

## Access Workflow

### CLI vs MCP tools

Use the structured Design Hub MCP tools for workspace and page work. Do not shell out to the `designhub` CLI for operations that have MCP tools available.

If the Design Hub MCP tools or namespace are not exposed in the current Codex turn, do not use the local `designhub` CLI as a fallback for workspace, view, TOC, page, draft, search, asset, auth, or profile operations. First report that the Design Hub MCP server is not available in the thread. If the user asks to diagnose plugin exposure, use the repo or installed CLI diagnostic command:

```bash
designhub mcp start
```

When the local HTTPS MCP server is running but a fresh Codex thread still does not expose the Design Hub MCP namespace, treat the remaining issue as Codex plugin runtime attachment or local streamable HTTP MCP configuration in the active Codex build. Route that issue to Codex/plugin runtime rather than changing Design Hub workspace operations.

The plugin MCP config must explicitly declare `"type": "streamable_http"` and `"url": "https://127.0.0.1:53113/mcp"` for the `design-hub` MCP server. If Codex cannot connect, confirm `designhub mcp start` is running and that the local HTTPS development certificate is trusted.

If plugin-provided MCP tools are still absent, use the documented Codex MCP config fallback:

Ask the user to explicitly enable or define the local Design Hub server in `~/.codex/config.toml`, then restart or reload Codex and start a fresh thread:

```toml
[plugins."design-hub@reimaginate".mcp_servers."design-hub"]
enabled = true

[mcp_servers.design_hub]
url = "https://127.0.0.1:53113/mcp"
```

If the direct `design_hub` server works but plugin-provided MCP does not, the local Design Hub MCP server is healthy and the remaining issue belongs to the Codex plugin runtime.

The CLI is still used by humans to configure authentication and start the MCP transport:

- MCP transport: `designhub mcp start`.
- Read-only MCP transport: `designhub mcp start --access read-only`.
- Create/update-only MCP transport: `designhub mcp start --access edit`.
- Workspace-scoped MCP transport: `designhub mcp start --workspace-id <workspace-id>`.
- Tenant-scoped MCP transport: `designhub mcp start --tenant-id <tenant-id>`.
- Human setup: `designhub login`. The default profile targets `https://api.designhub.online`; use `designhub setup --profile <name> --api-url <api-url>` only for additional environments.
- Human draft commands: `designhub drafts get`, `designhub drafts save`, `designhub drafts publish`, and `designhub drafts discard`.

The repository plugin configuration already uses this transport shape in `apps/design-hub/codex/plugin/.mcp.json`.

### MCP access restrictions

The local MCP server can be started with access restrictions. Treat these as hard server-side policy boundaries, not just preferences.

- In `--access read-only`, use only the read tools exposed by `tools/list`; writes are rejected by the server.
- In `--access edit`, create and update tools are available, but destructive tools such as `remove_toc_item`, `discard_draft`, and `update_page` with `allowDiscardDraft:true` are rejected.
- In `--access unrestricted`, all exposed Design Hub MCP tools are available, subject to normal Design Hub API authorization.
- When `--workspace-id` is set, do not call `list_workspaces`; use the configured workspace id and only call workspace-scoped tools for that workspace.
- When `--tenant-id` is set, omit `tenantId` or use the configured tenant id. Do not supply a different tenant id.
- If a tool returns `ACCESS_DENIED`, report the active restriction and ask the user to restart the MCP server with a broader mode only if the requested task genuinely requires it.

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

## TOC Identity

Design Hub TOC item IDs are not separate navigation IDs. A TOC item `id` is the underlying page or group ID:

- For page TOC items, `id` must equal the created page `id`, and `linkedSectionId` must also equal that same page `id`.
- For group TOC items, `id` is the group ID. A group does not normally have `linkedSectionId`.
- When adding anything under a group, use the group TOC item `id` as `parentId`.
- Do not invent a different TOC item ID for a page. Do not create page TOC items where `id` and `linkedSectionId` differ.
- Do not create a page TOC item as a placeholder for a section, heading, folder, or binder unless an actual Design Hub page has been created and populated for that purpose.
- If an existing TOC item violates this rule, do not copy the bad shape into new items; report the mismatch when it affects the requested work.

## TOC Groups

In Design Hub workspace navigation, a "group" means a table-of-contents group item.

When the user asks to create a group:

- Create a TOC item with `insert_toc_item`; do not create a page, workspace, folder, diagram group, security group, or data model group.
- Use a TOC item shaped as `type:"group"` and `label:<requested group name>`. If an `id` is supplied, that `id` is the group ID; otherwise use the server-returned item `id` as the group ID.
- Do not include `linkedSectionId` for a normal group unless the user explicitly asks to link or mirror TOC sections.
- Insert only one TOC item at a time. Do not submit a group with populated `children`; insert child pages or child groups separately.
- Use `parentId` only when the user asks for a nested group under an existing TOC group. Omit `parentId` for a root-level group.
- After insertion, use the returned TOC contents to find the new group id before adding children.

When the user asks to create a section, subsection, category, folder, or binder:

- Prefer a nested TOC group item when the item is only structural navigation.
- Create a populated introductory page only when the section needs readable content, such as purpose, audience, scope, links, status, owner, or how to use the child pages.
- If creating an introductory or binder page, first call `create_page` with useful content, then insert a page TOC item using `id:<created page id>` and `linkedSectionId:<created page id>`.
- Never represent a section with a page TOC item that has no real page behind it. Do not invent a page id to make a navigation item look like a page.
- If unsure whether the user wants a structural group or a readable introduction page, use a TOC group for the structure and create child pages for actual content.

When the user asks to create pages under a group:

- First call `get_toc` and find the named group. If it does not exist and the user's wording implies it should be created, insert the TOC group first.
- Create each requested page with `create_page`, then insert each page into the TOC with `insert_toc_item` using the group's TOC item id as `parentId`.
- Use a page TOC item shaped as `id:<created page id>`, `type:"page"`, `label:<created page title>`, and `linkedSectionId:<created page id>`.
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
- Only create a page TOC item for a real page returned by `create_page` or an existing page found with `get_page`, `list_pages`, or `search_pages`.
- Call `get_toc` for the workspace without `viewId`.
- Call `insert_toc_item` with `confirm:true`, `expectedRevision` from `get_toc`, `position:"append"`, and no `parentId`.
- Use a TOC item shaped as `id:<created page id>`, `type:"page"`, `label:<created page title>`, and `linkedSectionId:<created page id>`.
- If the TOC insert fails after page creation, report that the page was created but not placed in the TOC. Do not delete the page as cleanup.

For page updates:

- Call `get_page` first.
- Pass the returned `versionId` as `expectedVersionId`.
- Use `allowStaleVersion:true` only when the user explicitly accepts a stale overwrite.
- Do not discard drafts unless the user explicitly accepts that risk and the tool supports it.

Do not change workspace settings, view settings, permissions, production configuration, or unrelated pages unless explicitly requested.

## Page Content Authoring

Design Hub renders the page title from page metadata or Markdown front matter as the visible level 1 heading.

### Agent Guidance Discovery

Before creating or replacing substantial Design Hub page content, discover and apply relevant agent guidance.

- Read the root `AGENTS.md` and the nearest scoped `AGENTS.md` files that apply to the target repository, app, workspace, or content area.
- When `AGENTS.md` files conflict, prefer the closest applicable file over broader repository guidance.
- Treat `_Agents_.md`, `Agents.md`, or similarly named Design Hub workspace/page guidance discovered through TOC, page search, or adjacent content as local authoring guidance.
- Follow system, developer, and user instructions first; use repo `AGENTS.md` guidance for coding and repository conventions.
- Use Design Hub-local agent guidance for workspace/page writing style and structure, then sample existing pages to match the final section names, ordering, tone, and formatting.

### Style Matching Workflow

Before creating or replacing substantial Design Hub page content, sample the destination context so the new content matches the workspace's existing style and structure.

For page creation:

- Call `get_toc` and inspect the target parent group, neighboring pages, and labels before choosing the page structure.
- Read 2-3 sibling pages with `get_page` when the target group already has comparable pages.
- If useful siblings do not exist, use `search_pages` for a similar page type such as `implementation plan`, `architecture`, `decision`, `data model`, `workflow`, `sequence`, or `requirements`, then read the best matching page.
- Mirror the sampled pages' section names, section order, tone, front matter pattern, diagram placement, table style, and level-2 heading style.
- If no local style sample is available, use a concise Design Hub default: short opening context, level-2 section headings, business-friendly language, and no decorative boilerplate.

For page updates:

- Call `get_page` first and preserve the existing page's structure unless the user explicitly asks for a rewrite.
- Keep existing front matter, metadata, page title handling, section hierarchy, terminology, and diagram fences unless the requested change requires altering them.
- Make the smallest content change that satisfies the request; do not introduce unrelated sections or a new document template into an established page.
- When replacing most of a page, still follow the page's current pattern or the nearest sibling pattern instead of inventing a different structure.

Page type defaults when no stronger local sample exists:

- Architecture pages: `## Context`, `## Current State`, `## Target State`, `## Decisions`, `## Risks`.
- Implementation plans: `## Objective`, `## Approach`, `## Implementation Steps`, `## Validation`, `## Open Questions`.
- Data model or ERD pages: short context, `## Model`, a `yaml:erd` block, then `## Notes` when needed.
- Workflow or process pages: short intent, `## Flow`, a `diagram.workflow.v1.1.0` or `diagram.sequence.v1.1.0` block, then `## Operating Notes` when needed.
- Decision pages: `## Decision`, `## Context`, `## Options Considered`, `## Consequences`.

When creating or updating Markdown page content:

- Do not add a leading Markdown `# <page title>` heading when the page title is already supplied through `create_page`, `update_page`, draft title, or front matter.
- Start authored content with an introductory paragraph or a level 2 heading such as `## Purpose`, `## Audience`, or `## Scope`.
- Use level 2 headings as the top visible section headings inside normal Design Hub pages.
- Do not add an empty line immediately after Markdown headings. Put the following paragraph, list, table, or code fence on the next line because Design Hub heading styles already provide visual spacing.
- Do not add a blank line immediately before Markdown bullet lists. Keep bullet lists compact by starting them directly after the introducing paragraph or heading unless a specific Design Hub rendering need requires otherwise.
- If front matter includes `title:`, do not repeat the same title as a Markdown H1 after the front matter.
- If updating a page that already has both a front matter title and duplicate H1, remove the duplicate H1 when it is part of the requested content cleanup or when replacing the page body. Preserve it only when the user explicitly asks for a verbatim/minimal edit.
- Use a Markdown H1 only when the content will not have page title metadata or front matter rendered by Design Hub, or when the user explicitly requests an H1 in the body.

## Design Hub DSL Blocks

Design Hub pages use fenced Markdown code blocks for diagram DSL. Prefer the current Design Hub DSLs over Mermaid when they fit the request.

When creating or updating Design Hub pages for diagrams or executable workflows:

- Use `create_page` or `update_page` with Markdown content containing the relevant Design Hub DSL fenced code block.
- Use `diagram.workflow.v1.1.0` for Orchestrator workflow creation, automation specs, executable process definitions, or workflow implementation plans.
- Use `diagram.sequence.v1.1.0` for sequence diagram creation, integration flows, handoffs, service interactions, event timelines, and process conversations.
- Use `yaml:erd` for entity relationship diagrams, conceptual/logical data models, domain entities, table/entity structures, and relationships between records.
- Do not use Mermaid for workflow, sequence, or entity relationship diagram creation when the requested content fits one of these Design Hub DSLs.
- Put explanatory context, assumptions, owners, status, and links in normal Markdown around the DSL block; keep the fenced DSL block valid and focused on the diagram or workflow definition.

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

For detailed workflow grammar, use the repository guides:

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

For detailed sequence syntax, use `docs/sequence-diagram-user-guide.md`.

### Entity Relationship Diagrams

Use `yaml:erd` for entity relationship diagrams and data model diagrams. The current ERD implementation is `v1.0.0`.

Canonical fence:

````markdown
```yaml:erd
Title: Claims data model

entities:
  - entity: Worker
    alias: worker
    description: Worker master record
    pos:
      x: 0
      y: 0
    properties:
      - group: Keys
      - property: WorkerId
        description: Primary key

  - entity: Claim
    alias: claim
    description: Claim record
    pos:
      x: 500
      y: 0
    properties:
      - group: Keys
      - property: ClaimId
        description: Primary key
      - property: WorkerId
        description: Foreign key to Worker

relationships:
  - relationship: Worker has claims
    type: 1M
    from: Worker
    from_connector: right
    to: Claim
    to_connector: left
```
````

ERD rules:

- `Title` and `entities` are required; `relationships` is optional.
- Each entity needs `entity`, `alias`, `description`, and `pos` with numeric `x` and `y`.
- Entity `properties` may contain `group` rows and `property` rows; property rows need `description`.
- Relationship endpoints use entity names, not aliases.
- Relationship handles use `from_connector` and `to_connector` values such as `left`, `right`, `top`, or `bottom`.
- Cardinality labels are driven by `type`; use `11`, `M1`, `1M`, or `MM` when possible.
- Entity layout is manual; set `pos` values deliberately so entities do not overlap.
- Do not use Mermaid ER syntax inside `yaml:erd`.

For detailed ERD syntax, use `docs/erd-diagram-user-guide.md`.
