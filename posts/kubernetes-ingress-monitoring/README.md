---
title: "Kubernetes ingress monitoring from outside"
description: "Kubernetes probes never see your ingress, DNS, or load balancer. Check /readyz from outside the cluster, assert the body, and alert only on confirmed failures."
date: "2026-11-03"
validated: "2026-11-03"
# Target keyword: kubernetes ingress monitoring (secondary: external kubernetes uptime monitoring, monitor kubernetes endpoints from outside).
# Facts checked against content/docs/6-checks.md, 7-check-results.md, app/src/CheckForm.tsx, data_pipeline/engine/api/mcp.py and /pricing/ on 2026-11-03. See validation/kubernetes-ingress-monitoring.md.
# Do not add heartbeat checks, certificate-expiry alerts, or latency-threshold alerts unless the product ships them.
# MCP create_check/update_check take body_contains but not JSONPath assertions, headers, or auth. Keep the post saying those are set in the console.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in ScaledPapers".
---

Your pods are Ready. That does not mean anyone can reach them.

Kubernetes is good at checking itself. The kubelet runs liveness, readiness, and startup probes against each container, restarts the ones that hang, and takes the unready ones out of their Service. But those probes never leave the cluster. The kubelet calls the pod's IP directly, so a probe never touches the path your users take: public DNS, the cloud load balancer, TLS termination, the ingress controller, the ingress rule, and the Service in front of your pods.

That path is where a lot of "everything is green and nobody can reach us" outages happen:

- An ingress rule points at the wrong Service name or port after a rename. The ingress returns 404 or 503 while every pod is Ready.
- A TLS secret is replaced with the wrong certificate, or the controller falls back to its default self-signed one.
- The ingress controller itself is unhealthy, and every host behind it returns 502 or 504 at once.
- external-dns, or a manual change, leaves the public record pointing at an old load balancer.
- A security group or `loadBalancerSourceRanges` change blocks traffic from the internet while traffic inside the cluster keeps working.

More probes won't catch these. What catches them is one check per public host, run from outside the cluster, the way a customer's browser or API client reaches you.

## What to check from outside

**A readiness URL that goes through the ingress.** Expose a route such as `https://api.example.com/readyz` on the normal ingress path. Have it return a small JSON body only when the app can serve traffic. A 200 from that URL means DNS, TLS, the load balancer, the ingress rule, and the Service all worked for that request.

**The body, not only the status.** Proxies and default backends can answer 200 with an error page or the wrong app. Check for something only your app returns, such as a `status` field that equals `ok`. Match on the parsed field rather than raw text: `"status":"ok"` and `"status": "ok"` are the same JSON and different strings.

**The certificate.** Make the check fail on an invalid certificate. A wrong secret or the controller's self-signed fallback then shows up as a failure, not a green check behind a browser warning.

**The public DNS record.** If external-dns manages the record, or you move between load balancers, check that the name still resolves to what you expect. A record can resolve while the site behind it fails, and the reverse; see [DNS works, the site does not](/blog/dns-works-the-site-does-not/). For a one-off look at how a change has spread, the free [DNS propagation checker](/tools/dns-checker/) reads a record from 16 locations.

**Ports you expose without HTTP.** A `LoadBalancer` Service for Postgres, Redis, or MQTT can be checked with a TCP connect.

**More than one region.** A cloud or routing problem in one region looks like a full outage from there and like nothing from anywhere else. Checking from several regions tells you which one you have. See [why we check from different regions](/blog/why-we-ping-from-different-regions/).

## Alert on confirmed failures, not on rollouts

A rolling deploy, an HPA scale-up, or a node drain can drop a single request. That should not page anyone.

- **Wait for a pattern.** Alert after 2 or 3 failures in a row, not the first one. A real outage keeps failing; a rollout blip does not.
- **Set the timeout from real numbers.** Put it a little above your readiness endpoint's normal response time. A slow response that still works is not an outage.
- **Expect flapping.** A check that goes down, up, and down again within minutes should send one message, not one per change.

## What an outside check does not cover

- **CronJobs and background workers.** They have no public URL. The usual pattern is a heartbeat: the job pings a URL when it finishes, and you get an alert when the pings stop.
- **Certificate expiry ahead of time.** A failing check tells you the certificate is already invalid. To hear about it days before, keep cert-manager's own alerts on, and look up days remaining with the free [SSL certificate checker](/tools/ssl-checker/).
- **Internal Services.** Anything without a public endpoint stays with in-cluster monitoring: your probes and Prometheus alerts.

## A starting checklist

1. Add a `/readyz` route that goes through the ingress and returns JSON only when the app can serve.
2. Add one check per public host, with a check on a body field and certificate verification on.
3. Add one DNS check per hostname that external-dns manages or that moves between load balancers.
4. Add one TCP check per non-HTTP port you expose.
5. Alert after 2 or 3 failures in a row, with the timeout just above normal response time.
6. Put each public host on your status page as its own component.

Probes keep your pods healthy. A check from outside tells you whether anyone can reach them.

## Setting it up in ScaledPapers

The checklist works with any external monitor. Here is how each step maps to ScaledPapers checks, which run from outside your infrastructure. Saving a check rejects private, loopback, and non-http(s) targets, so use the public ingress hostname, not a cluster-internal name like `api.default.svc.cluster.local`.

### An API check for each readiness URL

Open **Checks → API**, click **Add check**, and enter `https://api.example.com/readyz`. API checks treat the expected status as an exact match, 200 by default.

Under **Assertions**, add **JSONPath equals**, for example `$.status` equals `ok`. **Body contains** also works, but it is an exact, case-sensitive text match, so it has the spacing problem above.

If the route needs a header or basic auth, add it under **HTTP request**. Leave **Verify SSL certificate** on, which is the default.

A Website check is looser: any status below 500 counts as up. That catches a dead controller, but not a 404 from a broken ingress rule. For a readiness URL, use an API check.

### A DNS check for the public record

Open **Checks → DNS** and add `api.example.com` with record type A, AAAA, or CNAME. Under **Assertions**, list the values the lookup must include. The check fails when the answer stops including them.

### A TCP check for non-HTTP ports

Open **Checks → TCP** and enter the load balancer's public hostname and port. You can send a payload and require **Response contains**, or set **Connect under ms** as a maximum connect time.

### Alerts, interval, and regions

**Confirmation period** is the number of failures in a row before the check is down (1, 2, 3, or 5; default 3), and **Recovery period** is the number of passes to come back up (1, 2, or 3). A single failure is stored and nothing is sent. If a check goes down again soon after it recovers, you get one message that it is flapping, and further down and up alerts pause until it settles. **Request timeout** is 5, 10, 15, or 30 seconds.

On Free, checks run every 5, 15, 30, or 60 minutes from 1 region, with email alerts. Pro adds 1- and 2-minute intervals, up to 5 regions (San Francisco, Ashburn, Montreal, Frankfurt, and Singapore), and Slack, Discord, PagerDuty, Pushover, and webhook alerts.

### Status page and incidents

Link each check to a component on a [status page](/free-status-page/). One check links to one component. Under **When the check is down**, pick **Create an incident automatically** if a confirmed failure should open an incident, or **Ask me to review first** to get a draft you approve. The default is **Don't create an incident**. See [Incidents](/docs/incidents/).

### Let your agent read the manifests

If your Ingress manifests live in the repo, an agent in Claude Code, Cursor, or Codex can do most of the setup over [MCP](/docs/mcp/). Ask it to read the Ingress hosts and health paths and create one check per host, with a **Body contains** assertion and a link to a component. JSONPath assertions, request headers, and HTTP auth are set in the console, since the MCP server leaves secrets there. The same API key works with the [REST API](/docs/api/) if you would rather do it from CI. More on the [MCP uptime monitoring](/mcp-uptime-monitoring/) page.

### What ScaledPapers does not do yet

There are no heartbeat checks for CronJobs, no warning before a certificate expires, and no alert on slow responses. Checks store each run's duration and show the average, and they alert on failures and timeouts.

When the ingress breaks and every pod is still Ready, your users should hear it from you. Start with the free plan: 5 checks and a status page, no card. See [pricing](/pricing/).
