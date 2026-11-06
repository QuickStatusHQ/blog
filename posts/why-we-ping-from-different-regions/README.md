---
title: Why we ping from different regions
description: A check from one city only answers “is it down for me?” Multi-region pings separate a global outage from a regional routing problem.
date: "2026-08-28"
validated: "2026-10-20"
# Structure follows docs/marketing/blog-post-structure.md: no product until "How the QuickStatus checker does it" (restructured 2026-11-03).
---

A website can look fine from your laptop and still be unreachable for customers on another continent. DNS, CDNs, and ISP peering all take different paths. One probe, from one place, cannot tell those cases apart.

## What a single location cannot see

A timeout from Virginia might mean the origin is down. It might also mean a bad path between that region and the host, a CDN POP that is unhealthy, or DNS that resolved to a dead address in one geography and a healthy one in another.

The useful question is rarely “is it down?” It is **where** it is down, and **for whom**.

## How to read a split result

Probe the same URL from several regions at the same moment, then read the pattern:

- **Every location fails** — treat it as origin or DNS. Walk those two apart in [DNS works, the site does not](/blog/dns-works-the-site-does-not/). The site is down for everyone.
- **One continent is red, the others are fine** — regional routing, a CDN POP, or an anycast hole. Customers in that region are the ones who will file tickets.
- **One city fails** — local to that path. Worth a second look, not an all-hands page.

Latency will always be higher from the far side of the world; that is expected. A jump from 90 ms to 400 ms in one city, with the others unchanged, is a different story than every city failing at once.

## Why not ping from the browser?

Your browser is one more single location, plus ad blockers, corporate proxies, and “mixed content” noise. Server-side probes from known regions keep the measurement comparable from one check to the next: same cities, same protocol, every time.

See the split before you guess. It tells you whether to fix the origin, open a ticket with the CDN, or wait out one bad route.

## How the QuickStatus checker does it

You can run that split on any URL right now. The free [website down checker](/website-down-checker/) hits the same URL from **16 AWS regions** at once and shows HTTP status and response time for each city:

**Americas** — Ashburn, Columbus, San Francisco, Portland, Montreal, São Paulo

**Europe** — Dublin, London, Paris, Frankfurt, Stockholm

**Asia Pacific** — Tokyo, Seoul, Osaka, Singapore

**Oceania** — Sydney

The cities stay the same from one run to the next, so two results are comparable.

When the failing city is where your customers are, tell them what's happening while you fix it: set up a [free status page](/free-status-page/).
