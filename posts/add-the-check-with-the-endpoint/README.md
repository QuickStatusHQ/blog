---
title: Add the check when you add the endpoint
description: "The health check usually gets added after the first outage. Ask your coding agent to add it in the same session it writes the endpoint, over MCP."
date: "2026-11-03"
validated: "2026-11-03"
# Target: uptime monitoring Claude Code / Cursor / Codex. Leaves "uptime monitoring MCP server" to /mcp-uptime-monitoring/ (unpublished); link it here once it is live.
# Link /blog/one-failed-check-is-noise/ once it is published.
# Facts from content/docs/14-mcp.md, data_pipeline/engine/api/mcp.py (server instructions, tool descriptions), 6-checks.md, /pricing/ on 2026-11-03.
---

A new endpoint ships. The health check for it gets added later, usually the week after it went down and nobody noticed.

Adding the check is not the hard part. You open a dashboard, copy a URL, fill in a form, and link it to the status page. It just happens in a different place, at a different time, from writing the code. So it waits.

If you write the endpoint with Cursor, Claude Code, or Codex, the agent that wrote it can add the check before the session ends.

## What the ask looks like

Connect the [ScaledPapers MCP server](/docs/mcp/), then ask in the same chat:

> Watch the health URLs in this repo and put each one on our status page as its own component.

The agent reads the repo for routes, so it already knows where `/health` and `/api/status` live. Over MCP it then:

1. Calls `usage` to see how many checks the plan allows.
2. Creates one check per public URL.
3. Creates the components on the status page and links each check to one.

The checks it creates are ordinary checks. They show up in the console and the [API](/docs/api/) like any other, so nothing the agent made is hidden from the rest of the team.

## Where it stops

**At the plan cap.** The server tells the agent to call `usage` before creating anything and to stop when a cap is reached. On the free plan that is 5 checks, and the API refuses a sixth either way. If the repo has nine health URLs, expect five checks and a note about the other four. You decide which five matter.

**At localhost.** Checks run from our probe regions, not your laptop. Private and loopback addresses are rejected. An agent working on a dev server will often reach for `http://localhost:3000/health` first. Give it the staging or production URL instead.

**At anything visitors see.** The server tells the agent to delete, open incidents, or email subscribers only when you asked for that action in the conversation. Those are instructions to the agent. The hard limits are in the next paragraph. A check set to **Ask me to review first** puts a draft incident under **Needs review** when it goes down. The agent can list those drafts. Publishing one emails subscribers and posts to Slack, so the agent only does it when you ask.

**At access.** These are enforced by the server, not left to the agent. Team invites and roles, creating and revoking API keys, billing, creating or deleting alert destinations, deleting an incident, and visitor subscribers stay in the console. The agent can attach and test an alert destination you already created. It cannot give itself more access.

## Setup

An owner or admin creates an API key on **API Keys** in the console and copies it once. Export it in the shell your editor uses:

```bash
export SP_API_KEY=sp_…
```

Install the skill and wire the server:

```bash
npx skills add ScaledPapers/skills --skill uptime-status-page -a claude-code -a codex -g -y
bash ~/.agents/skills/uptime-status-page/scripts/wire-mcp.sh claude   # or cursor, or codex
```

The script reads the key from the environment, so the raw key never goes into a project file. To add it by hand in Claude Code:

```bash
claude mcp add --transport http scaledpapers https://api.scaledpapers.com/mcp \
  --header "Authorization: Bearer ${SP_API_KEY}"
```

The Cursor and Codex snippets are in the [MCP docs](/docs/mcp/). The server has 34 tools. Cursor enables at most 40 MCP tools across all servers, so leave room for the others you use.

claude.ai and Claude Desktop custom connectors are not supported yet. They expect OAuth, and this server takes a bearer key.

## After the checks exist

The agent sets up the checks. The defaults decide when anyone gets paged. A new check needs 3 failed runs in a row before it counts as down, so one blip is stored and nothing is sent. If a check runs every 5 minutes and should page sooner, ask the agent to lower its confirmation period.

Maintenance works the same way. Over MCP, a maintenance window does not email subscribers unless you ask for it. Over the REST API, it emails them by default when a public component is included. Say so when you want the email sent.

---

API requests are unlimited and never billed on any plan. The free plan includes 5 checks and a status page. See [pricing](/pricing/).
