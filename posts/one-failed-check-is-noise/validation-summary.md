# Validation summary

Status
validated

Date: 2026-11-10

Host: api.quickstatus.dev

## What was checked

The timing loop in section 3 was run against QuickStatus's own readiness URL. Every claim about QuickStatus checks was compared with the docs and the engine source.

## 1. Timing loop

```
for i in $(seq 10); do
  curl -s -o /dev/null --max-time 20 -w '%{http_code} %{time_total}\n' https://api.quickstatus.dev/ready
done
200 0.922137
200 0.902744
200 0.881359
200 1.795402
200 2.851486
200 0.893066
200 1.876393
200 0.892631
200 0.871844
200 0.877351
```

Run from one client to the us-west-1 API on 2026-11-10. Median about 0.9 s, slowest 2.85 s. A 3-second timeout leaves almost no margin.

## 2. Time to alert table

Worst case ≈ interval × required failures: the first failing probe can land up to one interval after the outage starts, and each extra failure adds one interval. This assumes one region; see section 3.

## 3. QuickStatus claims

- **Confirmation period**: 1, 2, 3, or 5, default 3. **Recovery period**: 1, 2, or 3, default 1. **Request timeout**: 5, 10, 15, or 30 s. Source: `CONFIRMATIONS`, `RECOVERIES`, `TIMEOUTS` and defaults in `app/src/CheckForm.tsx`; `content/docs/6-checks.md` (Advanced).
- **One count across regions**: `apply_probe_result` in `data_pipeline/engine/repositories/outages.py` keeps one `consecutive_failures` / `consecutive_successes` pair per check (`EndpointState`), locked per endpoint, and resets failures to 0 on any passing probe. A region that keeps failing while others pass does not reach the confirmation count.
- **Samples**: status code, status text, **Checked from**, duration, date; filter by region. Source: `content/docs/7-check-results.md`.
- **Intervals**: Free 5, 15, 30, 60 minutes; Pro adds 1 and 2. **Regions**: Free 1; Pro up to 5 (San Francisco, Ashburn, Montreal, Frankfurt, Singapore). Source: `content/docs/6-checks.md`.
- **Alerts and flapping**: one down and one recovery alert; email on every plan, other destinations on Pro; flapping message inside a quiet window of at least 30 minutes, then alerts pause. Source: `content/docs/6-checks.md` (Alerts).
- **Incident hold**: interval × (confirmation + recovery), at least 5 minutes; a 5-minute check with defaults waits 20 minutes. Source: `content/docs/4-incidents.md`, `incidentHoldSeconds` in `CheckForm.tsx`.
- **Not available**: no quorum setting (the single counter above), no per-region alert, no latency alert.
