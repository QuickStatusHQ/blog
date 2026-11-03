# Validation summary

Status
validated

Date: 2026-11-03

## What was checked

Each claim about the MCP server, plan limits, and setup was compared with the engine source, the skill's wire script, the public docs, and the live MCP tool list.

## Result

- **Endpoint and auth**: `https://api.scaledpapers.com/mcp` with a bearer key. claude.ai and Claude Desktop custom connectors are not supported because they expect OAuth. Source: `content/docs/14-mcp.md`.
- **Tool count**: 34 `@mcp.tool(` registrations in `data_pipeline/engine/api/mcp.py`. The live server listed 34 tools on 2026-11-03.
- **Agent instructions**: the server tells agents to call `usage` before creating anything, to stop at a cap, and to delete, open incidents, or email subscribers only when the user asked in the conversation. Source: server `instructions` in `mcp.py`.
- **Free cap**: 5 checks, also enforced by the API (`plan_max_endpoints` in `data_pipeline/engine/api/routes/endpoints.py`). Plan values come from `billing.write_workspace_plan` and match `/pricing/`.
- **Private targets**: localhost, `.local`, private, loopback, and link-local addresses are rejected (`routes/endpoints.py`).
- **Review drafts**: `list_incident_proposals` and `approve_incident_proposal` exist. Approving publishes the incident, emails subscribers, and posts to Slack. The tool description says to run it only when the user asked.
- **Console only**: no MCP tool covers team, API keys, billing, creating or deleting alert destinations, deleting incidents, or visitor subscribers. `list_alert_destinations`, `test_alert_destination`, and `page_alert_destinations` work only on destinations that already exist.
- **Confirmation default**: 3 (`confirmation_failures` default in `EndpointIn`, `routes/endpoints.py`). MCP-created checks use the same model.
- **Maintenance email**: `maintenance_notify_value` returns false over MCP unless set. The REST API defaults to true when a public component is targeted (`content/docs/11-api.md`).
- **Setup**: `wire-mcp.sh` accepts `cursor`, `claude`, `codex`, or `all`. For Claude Code it registers the header as the literal `${SP_API_KEY}`, and for Cursor and Codex it writes environment references, so the raw key is not written by the script.
- **Billing**: "API requests are unlimited and never billed" is quoted from `/pricing/`.

## Not checked

- Cursor's 40-tool limit is Cursor's behavior, quoted from our MCP docs. It was not re-checked against Cursor.
- How an agent finds routes in a repo depends on the agent. The post describes what it is asked to do, not a guarantee.
