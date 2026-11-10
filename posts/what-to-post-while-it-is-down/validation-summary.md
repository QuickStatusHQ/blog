# Validation summary

Status
validated

Date: 2026-11-10

## What was checked

The templates in sections 1–5 are writing guidance and were not tested against a product. Every claim in *Setting it up in QuickStatus* was compared with the docs, the console, and the monitor function.

## Result

- **Incident fields**: Title, Status (investigating, identified, monitoring, resolved), Impact, at least one affected component, Message; message starts as *We are looking into the issue.* Source: `content/docs/4-incidents.md` (Create).
- **Impact labels**: Degraded performance, Partial outage, Major outage (`minor`, `major`, `critical` in the API). Only Major outage counts as major outage time on the 90-day bar. Source: `app/src/pages/Incidents.tsx`, `content/docs/4-incidents.md`. The 2026-10-20 draft said Minor / Major / Critical, which no longer matches the console.
- **Subscriber mail**: `notifySubscribers` in `supabase/functions/monitor/subscribers.ts` is called when an incident is created, when an update is posted, and when it is resolved. It sends to confirmed subscribers with all components or a matching component. RSS at `{publicUrl}/incidents.rss`; visitors follow it in Slack with `/feed subscribe`. Source: `content/docs/5-subscribe.md`.
- **Backfill**: **Backfill an incident that happened in the past** with start and end. Source: `content/docs/4-incidents.md`.
- **From a check**: **Create an incident automatically** or **Ask me to review first**; drafts are titled *{check} is down*, published with **Approve and publish**. Source: `content/docs/4-incidents.md` (Review before publishing), `content/docs/6-checks.md` (Status page).
- **Maintenance**: pauses linked checks, shows **Under maintenance** with the end time, is not counted as downtime, and open incidents list above it. **Notify subscribers** emails on scheduled, updated (time, duration, recurrence, weekdays, timezone, or scope), and cancelled windows, not on start or end. Source: `content/docs/13-maintenance.md`.
- **Not available**: no saved incident templates in the console; owner Slack (**Apps**) is *Coming soon*. Source: `content/docs/5-subscribe.md`.
