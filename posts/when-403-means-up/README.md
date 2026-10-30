---
title: When 403 means up
description: A WAF 403 is not an outage. A 200 with an error payload is. Scan HTTP status codes for website and API checks, and what to set as expected.
date: "2026-09-17"
---

HTTP status is a number on the wire. Up is a judgment. Treat every 4xx as down and you will page on bot walls, expired probe credentials, and rate limits. Treat every 200 as up and you will miss the health endpoint that returns `{"ok": false}`.

That is why ScaledPapers does not use one rule for every URL.

Website checks (and the free [website down checker](/website-down-checker/)): any HTTP status below 500 is up. API checks: exact match, default 200. Set this on [Add a check](/docs/checks/) or with `expected_status_codes` on the [Checks API](/docs/api/).

## Codes

| Code | Means | Website | API |
| --- | --- | --- | --- |
| Timeout | No response in time. Often one path, not a global outage. | Down | Down |
| DNS failure | Name did not resolve. HTTP never ran. Use a [DNS check](/docs/checks/) if you need to split “name” from “server.” | Down | Down |
| Connection refused | Nothing accepted the port. | Down | Down |
| 200 OK | Handler succeeded. Body may still be `{"ok": false}`. | Up | Up (default) |
| 201 Created | POST created a resource. Never 200. Do not create a row every minute unless you mean to. | Up | List it |
| 202 Accepted | Queued, not finished. | Up | List it |
| 204 No Content | Empty body (DELETE, HEAD, some health URLs). Skip body assertions. | Up | List it |
| 301 / 308 | URL moved for good. Stored status is the final hop if redirects are on. Update the check URL. | Up | List it, or change URL |
| 302 / 307 | Login wall or locale hop. Following can turn this into a 200 HTML login page. | Up | Turn redirects off |
| 304 Not Modified | Cache hit. Only if you sent `If-None-Match` / `If-Modified-Since`. | Up | List it |
| 400 Bad Request | Probe body, query, or `Content-Type` rejected. | Up | Fix the request |
| 401 Unauthorized | Missing or dead credentials. API may be fine. | Up | Fix auth |
| 403 Forbidden | WAF, bot score, allowlist, or not allowed. Same URL can be 200 in a browser. | Up | Down unless listed |
| 404 Not Found | Wrong path, or “no row.” A missing `/health` is a bad URL. | Up | Fix the URL |
| 405 Method Not Allowed | GET on a POST-only route, or HEAD with no HEAD. | Up | Change method |
| 408 Request Timeout | Server or proxy gave up waiting. | Up | Confirm, don’t one-shot |
| 409 Conflict | Duplicate create or version clash. | Up | Probe with GET |
| 410 Gone | URL retired on purpose. | Up | Update or delete the check |
| 413 / 415 / 431 | Body or headers too large or wrong type. | Up | Shrink the request |
| 422 Unprocessable | JSON parsed; business rules said no. | Up | Fix the payload |
| 429 Too Many Requests | You are probing too often, or a gateway is throttling. | Up | Slow the interval |
| 500 Internal Server Error | Origin handler crashed. | Down | Down |
| 502 Bad Gateway | Proxy got a bad upstream (CDN, load balancer, nginx). | Down | Down |
| 503 Service Unavailable | Overload, crashed workers, or planned maintenance. | Down | Down — or [schedule maintenance](/docs/maintenance/) |
| 504 Gateway Timeout | Proxy waited for upstream and gave up. One-region 504 is often a path, not a dead origin. | Down | Down |

Do not “fix” timeout, DNS, or refused with expected status. There is no code.

A lookup that returns an address, while the page still fails, is a later layer. Walk the port, the certificate, and the status in [DNS works, the site does not](/blog/dns-works-the-site-does-not/). A 502 on one path while the homepage is 200 is [Only some pages return 502](/blog/only-some-pages-return-502/).

A 301 to a parked page that finally returns 200 is the dangerous 3xx: redirects on, expected 200, origin gone. A chain that comes back to a URL it already returned is [Too many redirects](/blog/too-many-redirects/). A 503 with a friendly HTML body is still down.

## When 200 is the outage

Some APIs always return 200 and put the error in JSON: `{"error": "unavailable"}`. Add a JSONPath or body-contains assertion so a 200 with the wrong payload is down.

## What to set

1. Pick Website if “the host answered” is enough.
2. Pick API if clients depend on a specific code.
3. Set expected status to that code — 200, 201, 204, or a short list.
4. Add a body or JSONPath assertion when the status is always 200.
5. Turn follow-redirects off when a 302 login page would masquerade as 200.
6. Leave confirmation at three failures so a single 429 or 408 does not page.

The number on the response is data. The expected status is the contract.
