# Validation summary

Status
validated

Date: 2026-11-03

## What was checked

Every claim about QuickStatus checks was compared with the public docs, the console source, the MCP server, and `/pricing/`.

## Result

- **Plans**: Free is 10 checks, every 5, 15, 30, or 60 minutes, from 1 region. Pro adds 1- and 2-minute intervals and 5 regions. Source: `content/docs/6-checks.md` (Test frequency, Regions) and `/pricing/`.
- **Regions**: San Francisco, Ashburn, Montreal, Frankfurt, and Singapore. Source: `content/docs/6-checks.md`.
- **Public targets only**: saving rejects private, loopback, and non-http(s) targets; DNS rejects private, loopback, `.local`, and `localhost` names; TCP rejects private, loopback, and link-local targets. Source: `content/docs/6-checks.md`.
- **API vs Website**: API checks treat the expected status as an exact match (default 200). Website checks treat any response below 500 as up. Source: Assertions hints in `content/docs/6-checks.md`.
- **Assertions**: **Body contains** and **JSONPath equals** on API and Website checks. The text match is exact and case-sensitive. DNS expected record values; TCP **Response contains** and **Connect under ms**. Source: `content/docs/6-checks.md`.
- **Verify SSL certificate**: on by default (`useState(initial?.verify_ssl ?? true)` in `app/src/CheckForm.tsx`).
- **Request options**: HTTP authorization username and password, request headers, and a body. Source: `content/docs/6-checks.md` (HTTP request).
- **Confirmation and recovery**: confirmation 1, 2, 3, or 5 (default 3); recovery 1, 2, or 3; timeout 5, 10, 15, or 30 seconds. Source: `content/docs/6-checks.md` (Advanced).
- **Flapping**: a check that goes down again inside the quiet window after a recovery sends one flapping message and pauses further down and up alerts. Source: `content/docs/6-checks.md` (Alerts).
- **Alert destinations**: email on every plan; Slack, Discord, PagerDuty, Pushover, and webhook are Pro and attached per check. Source: `content/docs/6-checks.md` and `/pricing/`.
- **Status page**: one check per component; **When the check is down** defaults to **Don't create an incident**. Source: `content/docs/6-checks.md` (Status page).
- **MCP**: `create_check` and `update_check` accept `body_contains` and `verify_ssl`, but not JSONPath assertions, request headers, or HTTP auth ("Request headers, request body, and HTTP auth stay in the console"). Source: `data_pipeline/engine/api/mcp.py`.
- **Duration**: the check page shows **Test duration (avg)**. HTTP checks have no latency threshold alert. Source: `content/docs/7-check-results.md`.
- **No heartbeat checks**: the check kinds are API, Website, DNS, and TCP. Source: `content/docs/6-checks.md`.
- **Free tools**: the DNS propagation checker reads from 16 locations; the SSL certificate checker shows days remaining. Source: `public/tools/dns-checker/`, `public/tools/ssl-checker/index.html`.

## Not checked

- Kubernetes behavior (probes call the pod IP, the ingress controller's default certificate, 503 for a Service with no ready endpoints) is described from the Kubernetes and ingress-nginx documentation, not tested on a cluster for this post. Other ingress controllers may return different status codes.
