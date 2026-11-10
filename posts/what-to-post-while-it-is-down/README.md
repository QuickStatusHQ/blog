---
title: What to post while it is down
description: "Incident update templates for investigating, identified, monitoring, and resolved, plus a maintenance notice. What each update must say, and how often to post."
date: "2026-11-10"
validated: "2026-11-10"
# Target keyword: incident communication templates (secondary: status page incident update examples, outage update template, maintenance notification template).
# Facts checked against content/docs/4-incidents.md, 5-subscribe.md, 13-maintenance.md, app/src/pages/Incidents.tsx (impact labels) and supabase/functions/monitor (subscriber mail on create, update, resolve) on 2026-11-10. See validation/what-to-post-while-it-is-down.md.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in QuickStatus".
---

Customers forgive an outage they can follow. They do not forgive silence, and they do not forgive a planned deploy that looks like an accident.

The hard part is writing while the incident is still going: you do not know the cause, people are asking in five places, and the person who knows the most is busy fixing it. Templates help because the structure is decided before the outage starts.

## 1. What every update has to say

Four things, in the reader's words:

1. **What is broken.** "Checkout is failing", not "elevated 5xx on payments-svc".
2. **Who is affected, and what still works.** "New payments fail. Existing subscriptions and logins are fine." The second half saves you the most support tickets.
3. **What to do meanwhile.** Retry later, use the mobile app, do not resubmit the form. Leave it out if there is nothing.
4. **When the next update is.** A time, not "soon".

Put a timestamp in UTC (or the time zone your customers use) on every update.

Leave out:

- **Guesses about the cause.** "We think it is the database" becomes the story, even when it turns out to be DNS.
- **Internal names.** Customers do not know what `payments-svc` is.
- **Blame.** If a vendor is down, say so and link their status page. Customers still see your product failing.
- **"We apologize for any inconvenience."** Apologize once, at the end, and mean it.

## 2. The four updates

Most status pages use the same four stages. Here is one outage, written at each.

**Investigating.** Post this as soon as you know customers are affected, before you know why.

> Checkout is failing for new payments. Existing subscriptions are not affected. We are investigating. Next update by 14:45 UTC.

**Identified.** You know what is broken, even if it is not fixed.

> The payment provider is rejecting card charges from our servers. We have paused new charges so no card is charged twice. Logins and existing sessions still work. Next update by 15:15 UTC.

**Monitoring.** A fix is in. You are watching before calling it done.

> Card charges are going through again. We are watching error rates before we close this. If a payment failed between 14:10 and 14:41 UTC, it was not charged, and you can retry it now.

**Resolved.** It is over. Say what happened to the people affected.

> Payments are working normally. From 14:10 to 14:41 UTC, new card payments failed and were not charged. No other features were affected. We will publish a summary of the cause by Friday.

Only promise the summary if someone is going to write it.

## 3. How often to post

- **While investigating: every 30 minutes**, or sooner if anything changes. The next-update time is a promise. Keep it.
- **"No change" is an update.** "Still investigating, no change for customers, next update by 15:45 UTC" tells people you are still on it.
- **Post when the stage changes**, even if the last update was five minutes ago.
- **Once you are monitoring, slow down.** One update when you are confident, then resolve.

A long outage with regular updates reads better than a short one with no updates.

## 4. Planned maintenance

Planned work gets three notices, and none of them should surprise anyone.

**Before.** A few days ahead for anything customers will notice, longer for a large change.

> On Thursday 16:00–16:20 UTC we will restart the API database. The site stays up. Sign-in and saving may fail for up to five minutes during the window. Nothing needs to be done on your side.

**Start.**

> Maintenance has started. Sign-in may fail for a few minutes. We will post again when it is finished, by 16:20 UTC.

**Done.**

> Maintenance is finished. Everything is working normally.

If the work will overrun, post before the end time you promised, not after it.

## 5. Post in one place first

Pick one page as the source of truth and post there first. Then point at it from everywhere else: an in-app banner, the support auto-reply, a social post. Do not write a different version in each. Customers compare them.

The person writing updates should not be the person fixing the outage. They need ten minutes with the engineer every half hour, not the engineer's keyboard.

## Checklist

1. Every update says what is broken, who is affected, what still works, and when the next update is.
2. Post "Investigating" before you know the cause.
3. Post every 30 minutes while investigating, including "no change" updates.
4. The resolved update gives the start and end time and what affected customers should do.
5. Planned work gets a notice before, at the start, and at the end.
6. One page is the source of truth. Everything else links to it.

Customers can wait through an outage. They do not wait through silence.

## Setting it up in QuickStatus

The templates work on any status page. Here is how they map onto a QuickStatus page, where each update also reaches subscribers by email and RSS.

### Incidents

Open **Incidents → Create incident**. Fill in **Title**, **Status** (investigating, identified, monitoring, resolved), **Impact**, at least one **Affected component**, and **Message**. The message starts as *We are looking into the issue.* Replace it with your first update.

**Impact** is **Degraded performance**, **Partial outage**, or **Major outage**. The page shows the same words, and only Major outage counts as major outage time on the 90-day bar.

On an open incident, **Post an update** adds a new entry to the timeline with its own status and message. **Mark resolved** posts your final message. Each one is emailed to confirmed subscribers of the affected components and added to the page's RSS feed, which visitors can also follow in Slack. Visitors see the same timeline on the incident page.

To record an outage after the fact, check **Backfill an incident that happened in the past** and set the start and end.

### Incidents from a check

If a check is linked to a component, set **When the check is down** to **Create an incident automatically**, and a confirmed failure opens the incident for you. Or pick **Ask me to review first**: the failure puts a draft titled *{check} is down* under **Needs review**, and nothing is published until someone edits it and clicks **Approve and publish**. That is a good place to paste the Investigating template. Either way, the check does not write the updates for you. See [Incidents](/docs/incidents/).

### Maintenance windows

Open [Maintenance](/docs/maintenance/) and click **Add window**. Set the start, duration, and time zone, and select the check and its component together. During the window the check pauses, so the work does not open an incident, and the component shows **Under maintenance** with the end time. Maintenance time does not count as downtime, and an open incident still shows above it.

**Notify subscribers** is on when a public component is included. Subscribers are emailed when the window is scheduled, when its time or scope changes, and when it is cancelled.

### What QuickStatus does not do

There are no saved incident templates. Keep the ones above in your runbook and paste them in. Subscribers are not emailed when a maintenance window starts or ends, only when it is scheduled, changed, or cancelled. Posting incidents to your own Slack workspace from the console is coming soon. Visitors can already follow the RSS feed in Slack.

When the next outage starts, the first update should take a minute, not twenty. Set up a [free status page](/free-status-page/) now, while nothing is down.
