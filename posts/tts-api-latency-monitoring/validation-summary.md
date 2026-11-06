# Validation summary

Status
validated

Date: 2026-11-03

## What was checked

Every claim about QuickStatus checks was compared with the console and engine source, the plan migrations, and the live region list.

## Result

- **Plans**: Free is 10 checks, a 5-minute minimum interval, 1 region, and 14 days of results. Pro is 25 checks, 60 seconds, 5 regions, and 90 days. Source: `billing.write_workspace_plan` in `supabase/migrations/0049_resume_checks_on_pro.sql`, matching `/pricing/`.
- **Regions**: San Francisco (us-west-1), Ashburn (us-east-1), Montreal (ca-central-1), Frankfurt (eu-central-1), and Singapore (ap-southeast-1). Source: live `list_regions` on 2026-11-03.
- **Duration**: the HTTP probe starts its timer before the request and stops it after reading the response body, capped at 64 KB (`MAX_READ_BYTES = 65536` in `data_pipeline/engine/workers/checkers/http.py`). A one- or two-word clip is well under that cap.
- **Request options**: GET, POST, and the other methods. **Request headers**, a request body (disabled for GET and HEAD), **Expected status code**, and the **Body contains** and **JSONPath equals** assertions. Source: `app/src/CheckForm.tsx`.
- **Confirmation period**: 1, 2, 3, or 5 failures, default 3 (`CONFIRMATIONS` in `CheckForm.tsx`, `confirmation_failures` default in `tables.py`). **Request timeout**: 5, 10, 15, or 30 seconds (`TIMEOUTS`).
- **No latency alerts**: HTTP checks have no latency threshold. The post only uses the request timeout as the limit for slow responses.
- **Duration metric**: **Test duration (avg)** is an average (`app/src/pages/EndpointCheck.tsx`). Time ranges are capped at the plan's retention (`rangesForRetention`, `checkChart.ts`). The console shows no percentiles. `GET /v1/endpoints/{id}/history` returns `latency_ms` for each sample (`data_pipeline/engine/api/routes/endpoints.py`).
- **Incident option**: under **When the check is down**, the choices are **Don't create an incident** (default), **Create an incident automatically**, and **Ask me to review first** (`INCIDENT_MODE_CHOICES` in `CheckForm.tsx`).
- **Links**: `/is-it-down/elevenlabs.io` and `/is-it-down/deepgram.com` probe the provider's website (`probe_url` in `public/is-it-down/seed-targets.json`), and the post says so. `/tools/webhook-tester/`, `/docs/checks/`, `/pricing/`, and `/blog/why-we-ping-from-different-regions/` exist.

## Not checked

- Provider API endpoints, auth headers, and per-character billing. The post keeps these general on purpose.
- The character counts assume one 2-character request per run and a 30-day month.

This is a source check. No live synthesis request was sent to a TTS provider.
