---
title: Why we ping from different regions
description: A check from one city only answers “is it down for me?” Multi-region pings separate a global outage from a regional routing problem.
date: "2026-08-28"
validated: "2026-10-20"
---

A website can look fine from your laptop and still be unreachable for customers on another continent. DNS, CDNs, and ISP peering all take different paths. One probe, from one place, cannot tell those cases apart.

That is why the ScaledPapers checker does not ping from a single office. It hits the same URL from **16 AWS regions** and shows HTTP status and response time for each city. You can try it on the [free status checker](/).

## What a single location cannot see

A timeout from Ashburn might mean the origin is down. It might also mean a bad path between that region and the host, a CDN POP that is unhealthy, or DNS that resolved to a dead address in one geography and a healthy one in another.

The useful question is rarely “is it down?” It is **where** it is down, and **for whom**.

## The 16 cities

Checks run from four groups:

**Americas** — Ashburn, Columbus, San Francisco, Portland, Montreal, São Paulo

**Europe** — Dublin, London, Paris, Frankfurt, Stockholm

**Asia Pacific** — Tokyo, Seoul, Osaka, Singapore

**Oceania** — Sydney

Those cities match the landing checker. Latency will always be higher from the far side of the world; that is expected. A jump from 90ms to 400ms in one city, with the others unchanged, is a different story than every city failing at once.

## How to read a split result

- **Every city fails** — treat it as origin or DNS. Walk those two apart in [DNS works, the site does not](/blog/dns-works-the-site-does-not/). The site is down for everyone we can see.
- **One continent is red, the others are fine** — regional routing, a CDN POP, or an anycast hole. Customers in that region are the ones who will file tickets.
- **One city fails** — local to that path. Worth a second look, not an all-hands page.

The checker is there so you can see that split before you guess.

## Why not ping from the browser?

Your browser is one more single location, plus ad blockers, corporate proxies, and “mixed content” noise. Server-side probes from known regions keep the measurement comparable from one check to the next. The map is the product: same cities, same protocol, every time.

When the failing city is where your customers are, tell them what's happening while you fix it: set up a [free status page](/free-status-page/).
