---
title: One failed check is noise
seo_title: "False uptime alerts: one failed check is noise"
description: "A single timeout is not an outage. Require failures in a row, set the timeout above your slowest normal response, and read regions before you page anyone."
date: "2026-11-10"
validated: "2026-11-10"
# Target keyword: uptime monitoring false positives (secondary: false downtime alerts, alert fatigue uptime, how many failures before alert).
# Absorbs the held draft "One city fails" (2026-10-20); its reading rules live in post 1 and in section 4 here.
# Facts checked against content/docs/6-checks.md, 4-incidents.md, repositories/outages.py (one counter per check across regions) and live curl timings on 2026-11-10. See validation/one-failed-check-is-noise.md.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in QuickStatus".
---

A probe fails once. The next one passes. If that single failure paged someone, the monitor just taught the team to ignore it.

That is how real outages get missed. Nobody decides to ignore alerts. The channel fills with ones that were already over by the time anyone looked, people mute it, and the one that mattered arrives in the muted channel.

The fix is to separate what you record from what you alert on. Keep every failed request as data. Alert only when the failure has repeated.

## 1. Why one request fails when nothing is down

Most single failures are real failures of one request. They are just not outages.

- **The network path dropped a packet.** A lost SYN or a reset between the probe and your edge fails one request and nothing else.
- **The probe's resolver hiccupped.** DNS timed out on the monitor's side, not yours.
- **One response was slow.** A garbage-collection pause, a cold start, or a slow query pushed one request past a short timeout.
- **A deploy restarted a process.** A rolling restart drops the few connections that hit the old process as it exits.
- **Rate limiting caught the probe.** A 429 means the server answered and chose not to serve that client. See [When 403 means up](/blog/when-403-means-up/).
- **One region's route failed.** The site was up everywhere else. That is section 4.

None of these needs a human at 3 a.m. All of them are worth keeping in the history, because a pattern of them is a signal.

## 2. Count failures in a row

The rule most monitors use: the check is **down** after N failures in a row, and **up** again after M passes in a row.

The cost of a bigger N is time. The outage starts somewhere between two probes, so the worst case before the alert is about one interval for the first failure to land, plus one interval for each extra failure you require:

| Interval | 1 failure | 2 in a row | 3 in a row | 5 in a row |
| --- | --- | --- | --- | --- |
| 1 minute | ~1 min | ~2 min | ~3 min | ~5 min |
| 5 minutes | ~5 min | ~10 min | ~15 min | ~25 min |

Three in a row at a 1-minute interval alerts in about three minutes and ignores a single dropped request or a short restart. Three in a row at 5 minutes is fine for a marketing site and slow for checkout.

If the alert is too slow, shorten the interval before you lower N. A faster interval keeps the same protection against noise and gets you there sooner.

Retrying immediately inside one probe is a different tool. It catches a dropped packet. It does not catch a 30-second restart, because the retry lands inside the same 30 seconds. Failures spread across separate probes catch both.

## 3. Set the timeout above your slowest normal response

A timeout that is too short turns slow into down. Measure before you pick one:

```
for i in $(seq 10); do
  curl -s -o /dev/null --max-time 20 \
    -w '%{http_code} %{time_total}\n' https://api.example.com/ready
done
```

From a real readiness URL:

```
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

Most answers took 0.9 seconds. One took 2.85. A 3-second timeout would fail that URL on an ordinary day, from the same machine, with nothing broken. Set the timeout at several times the slowest normal answer, and run the loop at the busiest hour, not at night.

A timeout is not a latency alert. If you care that responses got slow, watch response time as its own signal. Lowering the timeout until slow responses fail just creates more single failures.

## 4. Read the regions before you call it an outage

When a check runs from several places, the split tells you what kind of failure it is.

- **Every region fails.** The origin, DNS, or the edge in front of it is down. This is the outage.
- **One region fails and the rest pass.** Customers on that path are affected. Look at the CDN location, geo rules, and routing before you announce a global outage.
- **One region is much slower than usual, and the others are unchanged.** That path is the problem.
- **Every region passes, and users still complain.** The check is not on the path they use: a login, a POST the check never sends, or a country you do not probe from.

More on reading the split in [Why we ping from different regions](/blog/why-we-ping-from-different-regions/).

## 5. Recovery and flapping

One pass is usually enough to call a check up again. Requiring more passes keeps a check that is barely working from looking healthy, at the cost of a later all-clear.

The harder case is a check that flaps: down, up, down, up, every few minutes. Each change is real, and alerting on every one is the same noise problem again. Better: send one message saying the check is flapping, hold the rest until it settles, then send one message when it has been up, or down, for a while.

## Checklist

1. Require 2 or 3 failures in a row before the check is down. Use 1 only when a single missed request is itself the outage.
2. If that alerts too slowly, shorten the interval. Do not drop to 1 failure.
3. Time 10 requests at a busy hour, and set the timeout at several times the slowest one.
4. Check from more than one region, and read the split before you page anyone.
5. Keep every failed sample. A failure at the same time every day is a cron job, a backup, or a deploy.
6. Send one flapping message, not one per state change.

The sample is data. The alert starts when the count is met.

## Setting it up in QuickStatus

Everything above applies to any monitor. Here is where each setting lives in QuickStatus, so the next single timeout gets stored and nobody gets paged.

### Confirmation, recovery, and timeout

Open a check and expand **Advanced**:

- **Confirmation period**: 1, 2, 3, or 5 failed checks in a row before the check is down. Default **3**.
- **Recovery period**: 1, 2, or 3 passes in a row to come back up. Default **1**.
- **Request timeout**: 5, 10, 15, or 30 seconds (**Connection timeout** on TCP checks).

A single failure is stored in the samples table with its status code, region, and duration. Nothing is sent.

On Free, checks run every 5, 15, 30, or 60 minutes. Pro adds 1- and 2-minute intervals, which is the faster way to shorten the time to alert from section 2.

### How regions count

Free checks run from 1 region. Pro runs from up to 5: San Francisco, Ashburn, Montreal, Frankfurt, and Singapore.

The confirmation count is one count per check, across all its regions. A pass from any region resets it. So when every region fails, the count fills quickly and the check goes down. When one region keeps failing and the others pass, the check stays up. Filter the samples by region on the check page to see the one that fails.

### Alerts and flapping

The account email gets one alert when the check goes down and one when it recovers. Slack, Discord, PagerDuty, Pushover, and webhook destinations are added under **Settings → Alerts** on Pro. If the check goes down again within a quiet window after recovering (at least 30 minutes), you get one message that it is flapping, and further down and up alerts pause until it settles.

### Incidents on a status page

Link the check to a component and set **When the check is down** to **Create an incident automatically**, or **Ask me to review first** for a draft you approve. A confirmed failure opens one incident. A short recovery does not close it: the incident resolves after the check has stayed up for its interval × (confirmation + recovery), and at least 5 minutes. A 5-minute check with the defaults waits 20 minutes. See [Incidents](/docs/incidents/).

### What QuickStatus does not do

There is no setting to require failures from 2 of 3 regions, and no alert for a single region on its own. There is also no alert on slow responses: durations are stored and averaged on the check page, and alerts fire on failures and timeouts.

When a real outage gets through the noise filter, tell customers before they ask. Set up a [free status page](/free-status-page/), or see [pricing](/pricing/) for 1-minute checks from 5 regions.
