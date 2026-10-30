---
title: DNS works, the site does not
description: "dig returned an address and the page still does not load. Test the port, then the certificate, then the HTTP status, and stop at the first failure."
date: "2026-10-20"
validated: "2026-10-20"
---

`dig` returned an address. The page still does not load. Leave the nameservers alone until you know a later layer is fine. The name did its job. Test the next layer, and stop at the first one that fails.

## 1. See whether every resolver returns the same answer

Run this from the machine where you are debugging. Use your hostname in place of `example.com`.

```
dig +short example.com A
dig +short example.com A @1.1.1.1
dig +short example.com A @8.8.8.8
```

If the name is supposed to be a CNAME, also run:

```
dig +short example.com CNAME
```

Read the three A lookups before you do anything else.

- **No address, or NXDOMAIN.** This is a DNS failure. HTTP never runs, so there is no status code to allow on a check. Fix the record or the delegation. The server is irrelevant until a lookup returns the address you expect.
- **The three lookups disagree.** Some resolvers still have the old address. Customers on those resolvers hit the old host, and a check that only asks “did it resolve?” stays green. The failure is the record, not the process.
- **They agree.** DNS is not why the site is down from here. Keep the address. You need it for the next step.

A CDN often returns several addresses, and they change. That is a normal answer, not a split. Do not write one of those addresses down as the only correct one. The check will go red every time the edge rotates.

If the site loads when you curl the IP and fails when you curl the name, you are in the opposite case. Step 1 was not fine. Stay on DNS.

## 2. See whether the port accepts a connection

Use the port you actually serve. For HTTPS that is 443.

```
nc -vz example.com 443
```

- **`succeeded` or `open`.** Something accepted a TCP connection. The certificate and the app are still untested. A successful connect is not a loaded page.
- **`Connection refused`.** A machine answered and refused the port. Nothing is listening, or the port is wrong. The process is down. Changing DNS will not start it.
- **It sits, then times out.** Packets are dropped. That is a firewall, a security group, or the path from this network. A timeout on your laptop is not yet “the origin is down.”

`nc` uses your path. Run the same URL on the [website down checker](/website-down-checker/) before you call it global. One city timing out, with the others fine, is that path. Every city failing is the origin or the name. How to read that split is in [Why we ping from different regions](/blog/why-we-ping-from-different-regions/).

## 3. See whether the certificate is valid for this name

```
curl -sI --max-time 15 https://example.com/
```

- **Response headers come back.** TLS worked. The status code on the first line is step 4. If the same URL appears twice in those headers, stop. That loop is [Too many redirects](/blog/too-many-redirects/).
- **The command stops with a certificate error.** Expired, unknown authority, or the name on the certificate does not match the host. The lookup pointed at a machine whose certificate is not valid for this site. A browser that loads the page only after a warning is not up.

Do not add `-k` and treat a success as the fix. `-k` tells curl to ignore the certificate. It hides the failure you just found.

A TCP check never does this step. It only opens the port. Certificate errors show up on an HTTP check with **Verify SSL certificate** left on.

## 4. Read the status code only after the certificate worked

The first line of that curl output is the first time HTTP matters. What each code means for a Website check versus an API check is in [When 403 means up](/blog/when-403-means-up/). Three outcomes show up constantly on this search:

- **502, 503, or 504.** Something accepted HTTPS. The app, or the proxy in front of it, failed. DNS and the port are fine. If the homepage is 200 and only one path is 502, compare those two URLs in [Only some pages return 502](/blog/only-some-pages-return-502/).
- **200, and the body says the app is down.** `{"status":"down"}` with a 200 is a green check unless you assert the body. Search for a string that appears only when healthy, such as `"status":"ok"`, or set a JSONPath assertion on `$.status`.
- **200 of the homepage, and customers still cannot log in.** You probed a cached page. Point the check at `/health` (or whatever path reaches the app) and have that handler send `Cache-Control: no-store`.

## Leave the same split running

The commands above are one afternoon. The outage after this one should not need them again.

1. Add a [DNS check](/docs/checks/) on the hostname, record type A or CNAME. If the address is stable, put the value from step 1 in the expected records, so a silent cutover goes red. If a CDN rotates the addresses, leave the expected list empty. “It resolved” is the DNS test, and the HTTP check answers “the right site responded.”
2. Add a [TCP check](/docs/checks/) on that host and port. Expected outcome: **Connection should succeed**. A timeout and a connection refused are both down, and the result says which. This check does not inspect the certificate.
3. Add an API or Website check on the URL customers open. Leave **Verify SSL certificate** on. That is the default, and it is the step `nc` cannot do.

Leave the confirmation period at three failures. One timeout is stored. It does not open an incident.

The red check is the layer you fix. The green ones are the layers you can stop blaming.
