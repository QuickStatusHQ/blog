---
title: When 403 means up
description: A WAF 403 is not an outage. A 200 with an error payload is. Scan HTTP status codes for website and API checks, and what to set as expected.
date: "2026-09-17"
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in ScaledPapers" (restructured 2026-11-03).
---

HTTP status is a number on the wire. Up is a judgment. Treat every 4xx as down and you will page on bot walls, expired probe credentials, and rate limits. Treat every 200 as up and you will miss the health endpoint that returns `{"ok": false}`.

So one rule for every URL does not work. A public website and an API your customers integrate with need different rules: for a website, "the host answered" is usually enough; for an API, clients depend on a specific code.

## Codes

| Code | Means | Is the service down? | What to do |
| --- | --- | --- | --- |
| Timeout | No response in time. Often one path, not a global outage. | Yes | Confirm from another region before calling it global |
| DNS failure | Name did not resolve. HTTP never ran. | Yes | Check DNS on its own to split “name” from “server” |
| Connection refused | Nothing accepted the port. | Yes | Check the port and the process behind it |
| 200 OK | Handler succeeded. Body may still be `{"ok": false}`. | No | Check the body too |
| 201 Created | POST created a resource. Never 200. | No | Expect 201. Do not create a row every minute unless you mean to |
| 202 Accepted | Queued, not finished. | No | Expect 202 |
| 204 No Content | Empty body (DELETE, HEAD, some health URLs). | No | Expect 204. Skip body checks |
| 301 / 308 | URL moved for good. A client that follows redirects ends at the final hop. | No | Update the URL you check |
| 302 / 307 | Login wall or locale hop. Following can turn this into a 200 HTML login page. | Usually not | Do not follow redirects when the hop matters |
| 304 Not Modified | Cache hit. Only if you sent `If-None-Match` / `If-Modified-Since`. | No | Expect 304 |
| 400 Bad Request | Probe body, query, or `Content-Type` rejected. | Not the service | Fix the request |
| 401 Unauthorized | Missing or dead credentials. The API may be fine. | Not the service | Fix auth |
| 403 Forbidden | WAF, bot score, allowlist, or not allowed. Same URL can be 200 in a browser. | Website: no. Your own API: yes, unless you expect it | Allowlist the probe, or expect 403 |
| 404 Not Found | Wrong path, or “no row.” A missing `/health` is a bad URL. | Not the service | Fix the URL |
| 405 Method Not Allowed | GET on a POST-only route, or HEAD with no HEAD. | Not the service | Change the method |
| 408 Request Timeout | Server or proxy gave up waiting. | Maybe | Confirm with more failures, do not alert on one |
| 409 Conflict | Duplicate create or version clash. | Not the service | Probe with GET |
| 410 Gone | URL retired on purpose. | Not the service | Update or delete the check |
| 413 / 415 / 431 | Body or headers too large or wrong type. | Not the service | Shrink the request |
| 422 Unprocessable | JSON parsed; business rules said no. | Not the service | Fix the payload |
| 429 Too Many Requests | You are probing too often, or a gateway is throttling. | Not the service | Check less often |
| 500 Internal Server Error | Origin handler crashed. | Yes | Fix the origin |
| 502 Bad Gateway | Proxy got a bad upstream (CDN, load balancer, nginx). | Yes | Check the upstream behind the proxy |
| 503 Service Unavailable | Overload, crashed workers, or planned maintenance. | Yes | If planned, announce maintenance first |
| 504 Gateway Timeout | Proxy waited for upstream and gave up. One-region 504 is often a path, not a dead origin. | Yes | Compare regions |

Do not “fix” timeout, DNS, or refused by expecting a status. There is no code.

A lookup that returns an address, while the page still fails, is a later layer. Walk the port, the certificate, and the status in [DNS works, the site does not](/blog/dns-works-the-site-does-not/). A 502 on one path while the homepage is 200 is [Only some pages return 502](/blog/only-some-pages-return-502/).

A 301 to a parked page that finally returns 200 is the dangerous 3xx: redirects followed, 200 expected, origin gone. A chain that comes back to a URL it already returned is [Too many redirects](/blog/too-many-redirects/). A 503 with a friendly HTML body is still down.

## When 200 is the outage

Some APIs always return 200 and put the error in JSON: `{"error": "unavailable"}`. Check the body, with a text match or a JSONPath on the field, so a 200 with the wrong payload is down.

## What to set

1. For a website, treat any answer below 500 as up, if “the host answered” is enough.
2. For an API, expect the exact code clients depend on — 200, 201, 204, or a short list.
3. Check the body when the status is always 200.
4. Do not follow redirects when a 302 login page would masquerade as 200.
5. Alert after three failures in a row, so a single 429 or 408 does not page.

The number on the response is data. The expected status is the contract.

## Setting it up in ScaledPapers

ScaledPapers checks build those two rules in, so you pick the rule when you pick the check kind.

- **Website checks** (and the free [website down checker](/website-down-checker/)): any HTTP status below 500 is up. Every code the table marks “No”, “Usually not”, or “Not the service” counts as up.
- **API checks**: exact match, default 200. List any other code you expect, such as 201, 202, 204, or 304, in **Expected status code**, or with `expected_status_codes` on the [Checks API](/docs/api/). A 403 on an API check is down unless you list it.
- **Body**: add a **Body contains** or **JSONPath equals** assertion for APIs that always return 200.
- **Redirects**: **Follow redirects** is on by default, and the stored status is the final hop. Turn it off when the hop itself matters.
- **Failures in a row**: **Confirmation period** defaults to three failures, so one 429 or 408 is stored and nothing is sent.
- **DNS and planned 503s**: a [DNS check](/docs/checks/) splits “name” from “server”, and a [maintenance window](/docs/maintenance/) announces planned downtime.

All of these are on [Add a check](/docs/checks/).

When the contract does break, tell users what's happening while you fix it: set up a [free status page](/free-status-page/).
