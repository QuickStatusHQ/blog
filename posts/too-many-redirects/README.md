---
title: Too many redirects
description: "Print the redirect chain and stop when a URL repeats. One hop from HTTP to HTTPS is normal. A loop never returns a status code."
date: "2026-10-20"
validated: "2026-10-20"
updated: "2026-11-18"
---

The browser says the page redirected you too many times. Two rules are sending the visitor back and forth. Print the chain, and stop at the first URL that appears twice.

## 1. Print the chain from a clean client

Run this from the machine where you are debugging. Use the URL that fails, including `http://` or `https://`.

```
curl -sI --max-time 20 http://example.com/
curl -sI -L --max-redirs 8 --max-time 20 http://example.com/
```

The first command is one response. The second follows up to eight hops and prints each status line.

The same chain is on the [redirect checker](/tools/redirect-checker/). It follows up to ten hops and shows the status, the time, and the headers on each one.

Read the `Location` header and the status lines before you clear a cookie or open a host panel.

- **A URL appears twice.** That pair is the loop. The later hops are the same two rules repeating. Fix those two. Do not keep reading.
- **The chain ends on a 200, and no URL repeats.** This search is the wrong one. A single hop from `http://` to `https://` is a normal redirect. What that final code means is in [When 403 means up](/blog/when-403-means-up/).
- **curl ends, and the browser still loops.** The loop is in the browser, usually a cookie or a stored HTTPS-only policy for that host. The server chain is fine. Stop. Clearing the site's cookies is the experiment, and editing the server will not change what curl just printed.

`curl` has no cookies unless you pass them. If it loops, a private window will loop too.

## 2. Name the two rules

The repeated pair is almost always one of these:

- **`http://` sends you to `https://`, and `https://` sends you back.** The edge accepted HTTPS and called the origin over plain HTTP. The origin saw HTTP and redirected to HTTPS. Each side is doing the thing it was told. They disagree about the scheme.
- **`www` sends you to the bare domain, and the bare domain sends you back to `www`.** Two hostname rules, usually in two different places: the host, and the edge.
- **A path sends you to another path that sends you back.** A trailing-slash rule against a framework rule is enough.

Change one side so both rules name the same next URL. After the edit, run step 1 again. The chain should end, and the URL that used to repeat should appear once.

A certificate warning is a different failure. Walk the port and the certificate in [DNS works, the site does not](/blog/dns-works-the-site-does-not/) when `curl` stops before any `Location` header.

## 3. Point the check at the page you meant

**Follow redirects** is on by default for an API or Website check. A chain that ends is judged by the last response. The 301 in the middle is not the stored status.

A loop never produces that last response. The probe errors, and the result has no status code. There is nothing to add to the expected-status list. The same is true of a timeout or a connection refused, which [When 403 means up](/blog/when-403-means-up/) already separates from HTTP codes.

Leave follow redirects on when you care about the page customers land on. Set the check URL to that page, the one step 1 ended on, so a later change to the hop is visible.

Turn follow redirects off when you care about the hop itself. A Website check then treats a 301 as up, because the default is any status below 500. An API check expects 200 unless you list 301. Both behaviors are in [Add a check](/docs/checks/).

Leave the confirmation period at three failures. One loop is stored. It does not open an incident.
