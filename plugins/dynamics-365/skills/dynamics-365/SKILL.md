---
name: dynamics-365
description: Use Dynamics 365 MCP tools to inspect Dataverse metadata, records, automation assets, solutions, users, roles, permissions, and security comparisons through the local D365 MCP server. Use when Codex needs live Dynamics 365 or Dataverse environment context. Do not use for generic Dataverse documentation, public Microsoft docs, or information already supplied in the conversation.
---

# Dynamics 365

Use the Dynamics 365 MCP tools when the user asks about live Dynamics 365 or Dataverse metadata, records, workflows, cloud flows, solutions, plugin assemblies, views, forms, option sets, users, roles, permissions, teams, or duplicate records in a configured environment.

Do not use these tools for public documentation lookups, generic Dataverse coding questions, repository-only inspection, or answers that can be completed from information already supplied in the current conversation.

## Access Workflow

### CLI vs MCP tools

Use the structured Dynamics 365 MCP tools for supported Dataverse environment inspection. Do not shell out to the `designhub d365` CLI for operations that have MCP tools available unless the user explicitly asks for a CLI command.

If the Dynamics 365 MCP tools or namespace are not exposed in the current Codex turn, do not use the local CLI as a fallback for Dataverse data, metadata, automation, solution, or security reads. First report that the Dynamics 365 MCP server is not available in the thread. If the user asks to diagnose plugin exposure, use the startup command:

```bash
designhub d365 mcp start
```

The plugin MCP config must explicitly declare `"type": "streamable_http"` and `"url": "https://127.0.0.1:33114/mcp"` for the `d365` MCP server. If Codex cannot connect, confirm `designhub d365 mcp start` is running and that the local HTTPS development certificate is trusted.

If plugin-provided MCP tools are still absent, use the documented Codex MCP config fallback:

Ask the user to explicitly enable or define the local Dynamics 365 server in `~/.codex/config.toml`, then restart or reload Codex and start a fresh thread:

```toml
[plugins."dynamics-365@reimaginate".mcp_servers.d365]
enabled = true

[mcp_servers.d365]
url = "https://127.0.0.1:33114/mcp"
```

The CLI is still used by humans to configure authentication, choose an environment profile, and start the MCP transport:

- MCP transport: `designhub d365 mcp start`.
- Profile-scoped MCP transport: `designhub d365 mcp start --profile <profile-name>`.
- Human setup: `designhub d365 setup --profile <profile-name> --url <dataverse-environment-url>`.
- Human login: `designhub d365 login --url <dataverse-environment-url>`.

### Profile safety

Codex must not manipulate Design Hub CLI or shared profile selection in any way.

Allowed:

- Use `auth_status` to inspect whether the already-selected D365 profile is usable.
- Tell the user to run `designhub d365 setup` or `designhub d365 login --url <dataverse-environment-url>` when profile target configuration or authentication is missing.

Forbidden:

- Do not create, update, switch, rename, remove, or select shared profiles.
- Do not change profile targets or the current profile.
- Do not pass `--profile` to `designhub` commands unless the user explicitly provided the profile for a diagnostic command.
- Do not run `profiles use`, `profiles set`, `profiles remove`, `profiles rename`, `profiles targets set`, `profiles targets unset`, or equivalent profile mutation commands.
- Do not edit local profile, token, credential, or connection string files directly.

If a task fails because the wrong profile is selected, a D365 target is missing, or the profile configuration is invalid, stop and report the exact issue. Ask the user to fix profile selection or profile configuration outside Codex, then retry after they confirm it is fixed.

## Tool Workflow

1. Start with `auth_status` before a task that needs Dynamics 365 access.
2. If profile target configuration is missing, tell the user to run `designhub d365 setup`.
3. If Dataverse authentication is missing, tell the user to run `designhub d365 login --url <dataverse-environment-url>` or sign in through the Azure Identity mechanism they use for the configured environment.
4. Use discovery tools before assuming logical names, ids, role names, workflow names, or user identities.
5. Keep reads scoped. Use limits and follow-up queries instead of broad environment dumps.
6. Do not request or expose secrets, connection strings, tokens, or credentials.

Common MCP tools:

- Authentication: `auth_status`.
- Metadata: `list_entities`, `describe_entity`, `list_fields`, `list_relationships`, `list_optionsets`, `list_views`, `list_forms`.
- Solutions and automation: `list_solutions`, `list_plugin_assemblies`, `list_entity_event_handlers`, `list_workflows`, `list_cloud_flows`, `export_workflow`, `export_cloud_flow`.
- Records: `get_record`, `query_records`, `count_records`, `find_duplicates`.
- Security: `list_users`, `list_roles`, `list_user_roles`, `list_user_permissions`, `compare_user_roles`, `compare_user_permissions`, `compare_user_teams`.

## Read Boundary

The Dynamics 365 MCP integration is intended for read, query, export, and comparison workflows. Treat retrieved Dataverse data as potentially sensitive business data.

Allowed:

- Inspect metadata, views, forms, option sets, solutions, plugin assemblies, workflows, and cloud flows.
- Retrieve a specific record by id when needed for the task.
- Query records with scoped FetchXML and reasonable limits.
- Count records, compare users, and find duplicate groups.
- Export workflow or cloud-flow definitions for analysis.

Forbidden:

- Do not create, update, delete, activate, deactivate, assign, copy, or otherwise mutate Dynamics 365 data or security through MCP.
- Do not broaden record queries beyond the user's task.
- Do not include credentials, access tokens, connection strings, or secret values in responses.
- Do not use destructive CLI commands as a substitute for unavailable MCP tools.

If a requested task requires a mutation that MCP does not expose, explain that the plugin is read-oriented and ask the user whether they want a separate CLI workflow.
