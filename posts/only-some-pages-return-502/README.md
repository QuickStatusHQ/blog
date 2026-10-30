---
title: Only some pages return 502
description: "Curl a URL that works and one that fails. Read both status lines. When the homepage is 200 and one path is 502, the proxy answered and that path's upstream did not."
date: "2026-10-20"
validated: "2026-10-20"
---

The homepage loads. Checkout, or one API route, shows 502 Bad Gateway. The name resolved, the port opened, and the certificate was accepted. Compare two URLs before you restart anything.

## 1. Curl the URL that works and the URL that fails

Run both from the machine where you are debugging. Use the path customers actually opened.

```
curl -sI --max-time 20 https://example.com/
curl -sI --max-time 20 https://example.com/checkout
```

The first command is the page that still loads. The second is the one that fails in the browser.

Read both status lines before you open a process manager or change DNS.

- **Either command exits before a status line.** Connection refused, timeout, or a certificate error means the problem is below HTTP for that request. Walk the name, the port, and the certificate in [DNS works, the site does not](/blog/dns-works-the-site-does-not/).
- **Both status lines are 502.** Every path you tried failed the same way from here. That is a whole-host failure from this client, not “one path only.” Use the same DNS post.
- **The homepage is 200 and the other URL is 502.** DNS, the port, and TLS worked for this host. The proxy in front answered on both requests. Only the backend for that path failed. Section 2 is where you stop.
- **The other URL is 404.** The server answered and said the path is missing. That is not a bad gateway. A missing page can still count as up on a Website check; see [When 403 means up](/blog/when-403-means-up/).
- **Both status lines are 200, and the browser still shows 502.** `curl` sent no cookies or session. The failure is tied to what the browser sends. Reproduce with the same cookies before you restart the app.

A **301** or **302** is a redirect, not a 502. Follow it with `curl -sI -L --max-redirs 8`, or run `curl` on the final URL, then read that status line. If a URL appears twice in the chain, stop in [Too many redirects](/blog/too-many-redirects/).

## 2. Read the 502 response

On the 502 response, the `Server` header names the proxy that gave up: Cloudflare, nginx, a load balancer. **502** means that proxy did not get a valid response from the process behind this path.

The homepage **200** means that proxy is running. Restarting the proxy does not start the upstream that only this route uses. Look at the app process, the timeout, or the route rule that sends this path to a different backend.

**504** is the sibling failure: the proxy waited and gave up on a slow upstream. The homepage still shows the proxy itself is up.

## 3. Monitor the path that failed

A check on `/` can stay green while `/checkout` returns 502. Add a Website or API check on the URL that returned 502. **502** is down for Website and API checks. Status-code rules are in [When 403 means up](/blog/when-403-means-up/); the form is [Add a check](/docs/checks/).

Leave the confirmation period at three failures. One 502 is stored. It does not open an incident.
