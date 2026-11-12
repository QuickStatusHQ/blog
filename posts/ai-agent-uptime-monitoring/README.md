---
title: "AI agent uptime monitoring from outside"
description: "An agent fails when its model API, MCP servers, or sandbox provider fails. Which endpoints to check from outside, how to check an MCP server, and what to skip."
date: "2026-11-03"
updated: "2026-11-12"
validated: "2026-11-12"
# Target keyword: AI agent uptime monitoring (secondary: MCP server uptime check, monitor AI agent dependencies, sandbox API monitoring).
# Inspired by the run-level sandbox monitoring angle in Northflank's "How to monitor and debug AI-agent sandboxes in production" (2026-09-14). This post covers the outside view only.
# Facts checked against content/docs/6-checks.md, 7-check-results.md, data_pipeline/engine/api/mcp.py, /pricing/, and a live initialize request to api.quickstatus.dev/mcp on 2026-11-03. See validation/ai-agent-uptime-monitoring.md.
# 2026-11-12: MCP server checks (initialize + tools/list), heartbeat checks, and slow alerts shipped (content/changelog.md 2026-11-12); console labels match the one-list Checks pages.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in QuickStatus".
---

An agent run can fail in two different places. Inside the run, the agent loops, calls the wrong tool, or writes the wrong file. That is a job for traces, logs, and run-level metrics. Outside the run, something the agent depends on stops answering: the model API, an MCP server, the sandbox provider, or your own API that starts runs. Run-level tooling sees the second kind only as a run that timed out or a tool call that errored. By then a user has already watched the agent fail.

This post covers the outside part. It is the part a check from the internet can answer: is each dependency reachable, does it answer correctly, and from where does it fail.

## List what the agent depends on

Write down every network call a run makes before it can do useful work. For most agents the list is short:

1. **The model API.** OpenAI, Anthropic, or your own inference server.
2. **Remote MCP servers** the agent uses as tools, whether yours or a third party's.
3. **The sandbox provider's API**, which creates the container or microVM the agent runs code in.
4. **Your own agent API**, the endpoint your app or customers call to start a run.
5. **Callbacks and webhooks** that receive results when a run finishes.

Each one needs its own check, because each one fails on its own.

## The model API

Send the smallest request your agent actually makes, with your own key, and check a field of the response. The provider's status page tells you what the provider has announced. It does not tell you whether your key, your model, and your region work right now. [Monitor the OpenAI API](/monitor-openai-api/) walks through that request step by step. The same request shape works for Anthropic, Groq, or a self-hosted model server with a public URL.

## MCP servers: a GET is not enough

A remote MCP server speaks JSON-RPC over HTTP POST. Many servers answer a plain GET with an error such as 405 Method Not Allowed. A monitor that treats any status below 500 as up reports that 405 as up, and keeps reporting up while the MCP handler behind the URL is broken.

Send a real MCP request instead:

- `POST` to the server's MCP endpoint, for example `https://mcp.example.com/mcp`
- Headers: `Content-Type: application/json`, `Accept: application/json, text/event-stream`, and `Authorization` if the server needs a key
- Body:

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"uptime-check","version":"1"}}}
```

- Expect status 200 and the text `protocolVersion` in the response

A server that is working replies with its capabilities and protocol version. Match on text rather than a JSON path: some servers reply as a `text/event-stream` instead of plain JSON, and a text match works on both.

This proves the server is up and answering MCP. It does not prove every tool works. Listing tools needs a session from the `initialize` reply, and a single HTTP request cannot carry that over. If one tool matters most, give it a plain HTTP health route of its own and check that too.

## The sandbox provider

Creating a sandbox on every check costs money and leaves cleanup behind if the check fails halfway. Check something cheaper: an authenticated read on the provider's API, such as listing your sandboxes or fetching your account. It proves the API is reachable and your key works.

For a full create, run, and delete test, run it yourself on a schedule inside your own system. Have it record when it last succeeded, and expose that on a public route such as `https://api.example.com/healthz/sandbox`. Return 200 when the last success is recent and 503 when it is not. Any outside check on that route now covers the whole round trip.

## Your own agent API and callbacks

Your agent API needs the same check as any other API: a health route that goes through your real load balancer and returns JSON only when it can start a run. Check the body, not only the status. If it runs on Kubernetes, the setup in [Kubernetes ingress monitoring from outside](/blog/kubernetes-ingress-monitoring/) applies as is.

Callbacks fail quietly. Check the receiving endpoint the same way, with a request it accepts without side effects.

## Alert once, from more than one place

Model APIs and MCP servers time out now and then. One slow call should not page anyone.

- **Wait for a pattern.** Alert after 2 or 3 failures in a row, not the first one.
- **Set the timeout from real numbers.** Put it above the dependency's normal response time. Model calls are slower than a web page, so 15 or 30 seconds is usual for the model API.
- **Check from more than one region.** If a provider fails from Frankfurt and works from Virginia, that is a routing or regional problem, not a global outage. A run in an EU sandbox and a run in a US sandbox can get different answers for the same reason.

## Show it on a status page

Users of an agent product see one thing: the agent did not finish. A status page can tell them why. Give each dependency its own component, for example *Agent runs*, *Model provider*, *Code sandbox*, and *Tools*, so a model outage does not read as "everything is broken".

## What an outside check does not cover

- **Run correctness.** Whether the agent did the right thing needs traces, run logs, and task validation inside the run.
- **Tokens, cost, and loops.** A check sees one request. It does not see a run that calls the same tool forty times.
- **Policy and security events.** Denied tool calls and unexpected outbound traffic belong to the sandbox and the agent framework.
- **Cleanup.** Whether a sandbox was deleted and its credentials revoked is a lifecycle check inside your system.
- **Runs that never report back.** The usual fix is a heartbeat: the job pings a URL when it finishes, and you get an alert when the pings stop.

## A starting list

1. A check on the model API, with a small real request and a check on a response field.
2. A check per remote MCP server: POST `initialize`, expect `protocolVersion`.
3. A check on the sandbox provider: an authenticated read, not a create.
4. A check on your agent API's health route, through the load balancer.
5. A canary run inside your system, exposed as a health route, if a full sandbox round trip matters.
6. Alerts after 2 or 3 failures in a row, and one status page component per dependency.

Run-level tooling tells you what the agent did. A check from outside tells you whether it could do anything at all.

## Setting it up in QuickStatus

The list above works with any monitor that can send a POST with headers. In QuickStatus, each MCP server is an MCP server check and everything else is an HTTP check.

**MCP servers.** Open **Checks**, click **Add check**, and pick **MCP server**. Enter the server URL and add the key under **Authorization**. Each run sends `initialize` and `tools/list`, and the check goes down if the server stops answering. List the tools your agent relies on under **Expected tools** and the check also fails when one disappears. Tools are listed, never called.

**Everything else.** Pick **Website or API** for the model API, the sandbox provider, and your own health routes, and set **Up when** to **Status is 200 OK**. Add the method, headers, and body under **HTTP request**. **JSONPath equals** works when the reply is plain JSON; **Body contains** is an exact, case-sensitive text match. Checks only accept public targets: private, loopback, and non-http(s) URLs are rejected, so a tool inside your VPC stays with your in-cluster monitoring.

**Alerts.** **Mark down after** sets how many failed checks in a row (1, 2, 3, or 5; default 3) make the check down. A single failure is stored and nothing is sent. **Request timeout** is 5, 10, 15, or 30 seconds. Email alerts are on every plan. Slack, Discord, PagerDuty, Pushover, and webhook alerts are Pro.

**Regions.** Free checks from 1 region. Pro checks from up to five: San Francisco, Ashburn, Montreal, Frankfurt, and Singapore, and the check page filters results by region.

**Status page.** Link each check to its component on a [status page](/free-status-page/). Under **When the check is down**, choose **Create an incident automatically**, or **Ask me to review first** if you would rather approve the wording. See [Incidents](/docs/incidents/).

**From your coding agent.** Claude Code, Cursor, or Codex can create and link these checks over the [uptime monitoring MCP server](/mcp-uptime-monitoring/). Request headers, request bodies, and HTTP auth are set in the console, since the MCP server leaves secrets there.

**Runs and slow answers.** For background runs, a **Heartbeat** check gives you a ping URL: call it when a run finishes (add `/fail` when it fails), and you get an alert when the pings stop. For a model API that answers but too slowly, set **Slow above (ms)** on the check, measured on total time or time to first byte. The check is marked **Slow**, not down, and you get an alert. For voice agents, where slow is its own failure, see [TTS latency monitoring](/blog/tts-api-latency-monitoring/).

When a dependency fails, your users should hear it from you before they open a ticket. Start with the free plan: 10 checks and a status page, no card. See [pricing](/pricing/).
