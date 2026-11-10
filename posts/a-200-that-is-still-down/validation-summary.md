# Validation summary

Status
validated

Date: 2026-11-10

Host: quickstatus.dev, api.quickstatus.dev

## What was checked

The commands in sections 1–3 were run against QuickStatus's own site and API. Every claim about QuickStatus checks was compared with the docs and the engine source.

## 1. Liveness and readiness

`data_pipeline/engine/api/main.py` defines `/health` as `return {"ok": True}` with no dependency, and `/ready` as a `select 1` against the database (plus a Redis ping when Redis is in use) before `{"ok": True}`. The post shows both handlers without the Redis branch.

```
curl -s -D - --max-time 20 https://api.quickstatus.dev/health
HTTP/2 200
content-type: application/json
server: uvicorn

{"ok":true}
```

## 2. Body assertions

```
curl -s --max-time 20 https://api.quickstatus.dev/health | jq -e '.ok == true'
true            (exit 0)

echo '{"status":"down","db":"timeout"}' | jq -e '.status == "ok"'
false           (exit 1)
```

`jq -e` exits non-zero on a false result, as the post says.

## 3. Cache headers

```
curl -sI --max-time 20 https://quickstatus.dev/
HTTP/2 200
age: 17139
cache-control: public,max-age=0,must-revalidate
cache-status: "Netlify Edge"; hit; ttl=31518861
server: cloudflare
cf-cache-status: DYNAMIC
```

Cloudflare did not cache the page (`DYNAMIC`); Netlify's edge served a stored copy (`hit`, `age` 17139 s). Two requests to `/blog/` a second apart returned `age: 1` (`fwd=miss; stored`) and `age: 950` (`hit`), so the age differs by edge location.

The meanings of `cf-cache-status` values (`HIT`, `STALE`, `DYNAMIC`), Always Online, `stale-if-error` (RFC 5861), and `Cache-Status` (RFC 9211) are from the Cloudflare docs and the RFCs, not reproduced with a failing origin.

## 4. QuickStatus claims

- **API vs Website**: API checks treat the expected status as an exact match (default 200). Website checks treat any response below 500 as up. Source: `content/docs/6-checks.md` (Check kinds, Assertions).
- **Assertions**: **Text to search for**, **Body contains**, **JSONPath equals**, **Status equals**. Text match is exact and case-sensitive. Source: `content/docs/6-checks.md`.
- **JSONPath**: a missing path fails (`jsonpath {path} not found`) and a different value fails (`jsonpath {path} != {wanted}`). Source: `data_pipeline/engine/workers/checkers/http.py`.
- **No header assertions**: the assertion kinds above are the full list.
- **Confirmation period**: 1, 2, 3, or 5, default 3 (`CONFIRMATIONS` in `app/src/CheckForm.tsx`).
- **MCP**: `create_check` accepts `body_contains`; JSONPath assertions stay in the console. Source: `data_pipeline/engine/api/mcp.py`, as in the Kubernetes post's report.
- **No latency alerts**: alerts fire on failures and timeouts; duration is stored and averaged. Source: `content/docs/7-check-results.md`.
- **Free plan**: 10 checks, a status page, email alerts. Source: `content/docs/6-checks.md` and `/pricing/`.
