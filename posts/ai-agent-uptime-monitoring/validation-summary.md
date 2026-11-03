# Validation summary

Status
validated

Date: 2026-11-03

## What was checked

Every claim about ScaledPapers checks was compared with the public docs, the MCP server source, and `/pricing/`. The MCP example was sent to a live MCP server.

## Result

- **MCP request**: `POST https://api.scaledpapers.com/mcp` with the `initialize` body in the post and `Accept: application/json, text/event-stream` returned HTTP 200 and a JSON-RPC result containing `"protocolVersion":"2025-06-18"` on 2026-11-03. A plain `GET` to the same URL returned 405.
- **Website vs API**: Website checks treat any response below 500 as up, so a 405 counts as up. API checks treat the expected status as an exact match (default 200). Source: Assertions hints in `content/docs/6-checks.md`.
- **Request options**: POST, request headers, HTTP authorization, and a request body (disabled for GET and HEAD). **Body contains** is an exact, case-sensitive text match; **JSONPath equals** is also available. Source: `content/docs/6-checks.md`.
- **Public targets only**: saving rejects private, loopback, and non-http(s) targets. Source: `content/docs/6-checks.md`.
- **Confirmation period** 1, 2, 3, or 5 (default 3); **Request timeout** 5, 10, 15, or 30 seconds. Source: `content/docs/6-checks.md` (Advanced).
- **Regions and plans**: Free is 5 checks from 1 region; Pro is up to 5 regions (San Francisco, Ashburn, Montreal, Frankfurt, Singapore). Email alerts on every plan; Slack, Discord, PagerDuty, Pushover, and webhook alerts on Pro. Source: `content/docs/6-checks.md` and `/pricing/`.
- **Incidents**: **When the check is down** offers **Create an incident automatically** and **Ask me to review first**. Source: `content/docs/6-checks.md` (Status page).
- **MCP server limits**: `create_check` cannot set request headers, request body, or HTTP auth. Source: `data_pipeline/engine/api/mcp.py`.
- **Duration**: the check page shows **Test duration (avg)**; there are no latency threshold alerts. Source: `content/docs/7-check-results.md`.
- **No heartbeat or MCP check type**: check kinds are API, Website, DNS, and TCP. Source: `content/docs/6-checks.md`.

## Not checked

- Third-party MCP servers. Some reply as `text/event-stream`, and some require a session or auth before `initialize`; the post tells readers to use Body contains and add auth.
- Sandbox provider APIs. The post describes an authenticated read in general terms and does not name endpoints.
