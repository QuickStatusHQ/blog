---
title: A 200 that is still down
seo_title: "Health check returns 200 but the site is down"
description: "A health check can return 200 while the database is down or a CDN answers for a dead origin. Read the body, read the cache headers, and make /ready honest."
date: "2026-11-10"
validated: "2026-11-10"
# Target keyword: health check returns 200 but service is down (secondary: health check body assertion, cached 200 origin down, liveness vs readiness endpoint).
# Merges the held drafts "A 200 that is still down" and "The cache answered" (2026-10-20). /blog/the-cache-answered/ was never published.
# Facts checked against content/docs/6-checks.md, workers/checkers/http.py (jsonpath), data_pipeline/engine/api/main.py and live curl runs on 2026-11-10. See validation/a-200-that-is-still-down.md.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in QuickStatus".
---

HTTP 200 means the handler returned a success code. It does not mean login works, the database answered, or the JSON says ok.

A monitor that only reads the status line stays green in three cases where users are already failing:

- **The health URL checks nothing.** The handler returns 200 as long as the process is running. The database can be gone.
- **The failure is in the body.** The app always returns 200 and writes `{"status":"down"}` or `{"ok":false}`.
- **A cache answered.** A CDN or reverse proxy served a stored copy, and the origin behind it is dead.

[When 403 means up](/blog/when-403-means-up/) covers which status codes count as up. This post is the next question: what the 200 actually checked.

## 1. Find out what the health URL touches

Request the health URL and print the headers and the body together:

```
curl -s -D - --max-time 20 https://api.example.com/health
```

Then open the handler. Most frameworks generate a health route that only proves the process is alive. Here is one from a real API:

```
@app.get("/health")
async def health():
    return {"ok": True}

@app.get("/ready")
async def ready():
    async with SessionLocal() as session:
        await session.execute(text("select 1"))
    return {"ok": True}
```

`/health` returns `{"ok":true}` while the database is down. `/ready` runs a query first, so it fails when the database does. Kubernetes uses the same split: **liveness** asks whether to restart the process, **readiness** asks whether it can serve traffic. An outside check wants the readiness answer.

If the only health URL is the liveness kind, an outside check on it watches the process and nothing behind it.

## 2. Read the body, not only the status

When the app puts its verdict in the body, check the body.

`jq -e` exits non-zero when the expression is false or the field is missing, so it works in a script or a CI step:

```
curl -s --max-time 20 https://api.example.com/ready | jq -e '.status == "ok"'
```

A plain text match is easier to get wrong. The letters `ok` also appear in `token`, `broken`, and `"ok": false`. And `"status":"ok"` does not match `"status": "ok"` with a space. If you must match text, pick a string that only appears when the service is healthy, and copy it from a real response instead of typing it.

```
curl -s --max-time 20 https://api.example.com/ready | grep -qF '"status":"ok"'
```

A structured check on a field (`$.status` equals `ok`) avoids both problems. It also fails when the field is missing, which is what you want when someone renames it.

## 3. Find out whether a cache answered

Ask for the headers of the URL your monitor hits:

```
curl -sI --max-time 20 https://example.com/
```

From a site behind Netlify and Cloudflare:

```
HTTP/2 200
age: 17139
cache-control: public,max-age=0,must-revalidate
cache-status: "Netlify Edge"; hit; ttl=31518861
server: cloudflare
cf-cache-status: DYNAMIC
```

The headers that matter:

- **`Age`** is how many seconds ago the stored copy was fetched from the origin. `17139` is nearly five hours. A large `Age` on a health URL means the origin was not asked.
- **`Cache-Status`** (RFC 9211) names each cache and whether it was a `hit`. Netlify, Fastly, and others send it.
- **`cf-cache-status`** is Cloudflare's version. `HIT` came from cache. `STALE` came from cache after Cloudflare could not reach the origin to refresh it, which on a health URL means the origin is in trouble. `DYNAMIC` means Cloudflare did not cache the response, though a cache behind it may have, as above.
- **`X-Cache`** is the older form some CDNs and proxies still send.

Two edge features return 200 on purpose when the origin fails. A `Cache-Control` with `stale-if-error` lets a cache serve an old copy instead of an error. Cloudflare's **Always Online** serves a saved copy of a page when the origin is unreachable. Both are good for visitors reading a blog post. Both hide an outage from a monitor pointed at the same page.

If you are not sure, request the same URL twice and compare `Age`. A number that keeps growing came from cache.

## 4. Make the health URL tell the truth

The fix is in the handler, not the monitor.

1. **Touch the dependencies that matter.** Run one cheap query against the database. Ping the queue or cache if requests cannot be served without it. Give each a short timeout, shorter than your monitor's.
2. **Return 503 when a required dependency fails.** Then the status code alone is enough, and every monitor, load balancer, and orchestrator understands it.
3. **Send `Cache-Control: no-store`.** No shared cache should keep a copy of a health answer.
4. **Leave optional dependencies out of the verdict.** If search being down still lets people log in and pay, report it in the body (`"search":"degraded"`) and keep the status 200. Otherwise one slow add-on makes the whole service look down.
5. **Keep it cheap and unauthenticated.** It will be called every minute from several places. It should not need a token, and it should not run the expensive query.
6. **Keep the liveness route too.** Something should still be able to ask "is the process alive" without paging on the database.

If you cannot change the handler, keep the 200 and assert the body, as in section 2.

## Checklist

1. Point the check at a URL that touches the database, such as `/ready`, not `/` or a liveness-only `/health`.
2. Return 503 from it when a required dependency is down, and send `Cache-Control: no-store`.
3. If it can only ever return 200, assert a field in the body, not just the status.
4. Run `curl -sI` on the URL you monitor and read `Age`, `Cache-Status`, and `cf-cache-status`. A health URL should never be a cache hit.
5. Keep a separate check on the homepage if you also care that the page loads. Read that one as reachability, and the health URL as whether the app works.

The status code is the envelope. The body and the headers say what is inside.

## Setting it up in QuickStatus

The checklist works with any outside monitor. Here is how to leave the same checks running in QuickStatus, so the next 200-but-down is caught by a check and not by a customer.

### An API check on the readiness URL

Open **Checks → API**, click **Add check**, and enter the readiness URL, for example `https://api.example.com/ready`. API checks treat the expected status as an exact match, 200 by default, so a 503 from the handler is a failure.

Use an API check here, not a Website check. A Website check counts any status below 500 as up, which is right for a homepage. It also means a 200 with `"status":"down"` in the body is up until you add an assertion.

### Assert the body

Under **Assertions**:

- **JSONPath equals** checks one field, for example `$.status` equals `ok`. A missing path fails, and so does a different value.
- **Text to search for** (or **Body contains**) is an exact, case-sensitive text match, with the spacing problem from section 2.

If you add checks from Claude Code, Cursor, or Codex over [MCP](/docs/mcp/), `create_check` accepts a body-contains string. JSONPath assertions are set in the console.

### Keep one failure from paging

**Confirmation period** is how many failed checks in a row it takes before the check is down: 1, 2, 3, or 5, default 3. A single failed request is stored and nothing is sent. [One failed check is noise](/blog/one-failed-check-is-noise/) covers how to pick the number.

### What QuickStatus does not check

Assertions read the status code and the body. There is no assertion on response headers, so a check cannot fail on `Age` or `cf-cache-status`. Point it at a URL that sends `no-store` instead. There is also no alert on slow responses: each run's duration is stored and averaged on the check page, and alerts fire on failures and timeouts.

When the health URL says ok and the app is down, the people who notice first are your users. Start with the free plan: 10 checks, a status page, and email alerts, no card. See [pricing](/pricing/).
