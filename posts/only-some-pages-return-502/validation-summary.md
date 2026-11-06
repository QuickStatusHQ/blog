# Validation summary

Status
validated

Date: 2026-10-20

Host: quickstatus.dev

## What was checked

The post opens with two `curl -sI` commands: one URL that loads, one that does not. These runs use the homepage, a blog article, and a path that does not exist on this host.

## 1. Homepage and a blog path

```
curl -sI --max-time 20 https://quickstatus.dev/
HTTP/2 200
server: cloudflare

curl -sI --max-time 20 https://quickstatus.dev/blog/too-many-redirects/
HTTP/2 200
server: cloudflare
```

Both status lines are HTTP/2 200. Cloudflare answered for each URL.

## 2. Path that does not exist

```
curl -sI --max-time 20 https://quickstatus.dev/no-such-path-502-check
HTTP/2 404
server: cloudflare
```

The status line is HTTP/2 404. The post treats that as a missing page, not a bad gateway.

Branch: homepage and blog path are 200; the missing path is 404. No URL returned 502.
