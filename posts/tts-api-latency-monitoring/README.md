---
title: "TTS latency monitoring: slow or down?"
description: "Your text-to-speech API can be up and still too slow for a voice agent. How to watch TTS uptime and latency from outside, and what to alert on."
date: "2026-11-03"
validated: "2026-11-03"
# Target keyword: TTS latency monitoring (secondary: TTS API monitoring, text to speech API uptime).
# Facts checked against app/src/CheckForm.tsx, EndpointCheck.tsx, checkChart.ts, workers/checkers/http.py, routes/endpoints.py, plan migrations and live list_regions on 2026-11-03. See validation/tts-api-latency-monitoring.md.
# Do not add time-to-first-audio measurement, percentiles in the UI, or latency-threshold alerts unless the product ships them.
# build-blog.mjs does not emit FAQPage schema; the FAQ below is plain content.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in ScaledPapers" (restructured 2026-11-03).
---

A text-to-speech API fails in two ways. The obvious one is **down**: requests time out or come back 5xx. The quieter one is **slow**: the API still answers 200, but the first audio arrives a second late and your voice agent sounds like it froze. Callers hang up either way.

The provider's status page tells you what the provider has confirmed. A check from outside tells you what your requests are getting right now.

## What "down" looks like for a TTS API

For ElevenLabs, Deepgram, OpenAI's speech endpoint, or any other voice API, "down" usually shows up as one of these:

- Connection or TLS failures to the API host
- 5xx responses from the synthesis endpoint
- 429 responses when your own traffic is normal
- 401 or 403 after a key rotation. That one is on your side, but production breaks the same way.
- A 200 with an empty body or an error in the JSON

A ping of the provider's marketing site catches none of these. The marketing site and the API usually run on different hosts. You need a check against the API itself.

## What "slow" looks like

Speech latency has more than one clock:

- **Time to first byte**: when the response headers arrive. Easy to measure, but on a streaming endpoint headers can arrive well before any audio.
- **Time to first audio**: when the first playable chunk arrives. This is what a caller hears.
- **Total synthesis time**: when the whole clip is done. Matters for batch jobs, less for streaming agents.

Most HTTP monitors measure total time: from the start of the request until the body has been read. Keep the synthesis request tiny, one or two words, and total time comes close to time to first audio. It moves when the provider gets slow.

Look past the average. A 300 ms average can hide one request in twenty that takes 2.5 s or longer, and that is the one a caller notices.

## Two checks that cover most of it

You do not need to synthesize audio every minute to know the API is reachable. Two checks do most of the work.

**1. Reachability check (frequent, no synthesis cost).** A GET against a lightweight authenticated endpoint, such as the provider's voices or models list, with your key in the header the provider expects. Expect 200, and check the body too, so an empty or error body counts as a failure.

**2. Synthesis canary (less often, real audio).** A POST to the synthesis endpoint with `Content-Type: application/json` and a body with very short text such as `OK`. Expect 200. The response is audio, so check the status code only. A text match cannot read binary audio.

Run the canary every 15 minutes, or every 5 if voice is the core of your product. Do the math before you pick: `OK` every 5 minutes is about 17,000 characters a month per region, and every 15 minutes is about 6,000. Check how your provider bills very short requests. Use a separate key with a spending cap, not a low rate limit. A tight rate limit can produce the same 429s you are watching for. A separate key also means you can rotate it without touching production.

The same two-check setup works for an LLM API. See [monitor the OpenAI API](/monitor-openai-api/).

### Regions

From one region you know whether the API answers from there. From several, a failure in one city while the rest succeed usually means routing near that city, not a provider outage. That is the difference between paging someone and writing a note. More on that in [why we ping from different regions](/blog/why-we-ping-from-different-regions/).

## What to alert on

**Failures in a row, not one failure.** Alert when several checks in a row fail. On a canary that runs every 15 minutes, 3 failures is half an hour, so use 2. On a 5-minute reachability check, 3 is fine.

**Slow as a failure.** A request timeout is the hard limit. Set it to the latency your agent cannot live past, not your normal latency. A 5-second timeout on the canary turns any synthesis slower than that into a failed check, and the failures-in-a-row rule still applies before anyone is paged.

**Read the status code before you blame the provider.** A 401 or 403 fails a check the same way an outage does. Look at the code before you escalate. If it is 401, it is your key.

**Latency trend.** Watch the duration per region over days, not minutes. Look again when the provider ships a new model or moves regions. Compare p50 and p95, not only the average.

## Tell your users before they tell you

If your product speaks, your customers notice a TTS outage quickly. Give voice its own component on your status page, for example "Voice responses", and open an incident on it when the canary is confirmed down. Then post one line:

> Voice replies are delayed because of an upstream provider issue. Text replies still work.

## Quick checks when something feels off

- **Is the provider's site reachable at all?** See [elevenlabs.io](/is-it-down/elevenlabs.io) or [deepgram.com](/is-it-down/deepgram.com) from 16 locations. That page checks the provider's website, not its API, so it backs up your own check and does not replace it. Read the provider's status page too.
- **Is the problem your webhook or callback, not the synthesis?** Replay the payload in the [webhook tester](/tools/webhook-tester/).

## FAQ

**Can I monitor a TTS API without paying for synthesis every minute?**
Yes. Check a free authenticated endpoint, such as the voices or models list, often. Run a tiny synthesis request less often.

**What is a good latency target for a voice agent?**
It depends on the provider and model. Watch the duration of your canary for a week from the regions your users are in. Then set the request timeout above the slowest runs you can accept.

**Should I monitor from one region?**
One region tells you whether the API answers from there. It cannot tell a provider outage from a network problem near the probe. That takes several regions.

Down is easy to see. Slow is what callers hear first, so measure it from where they are.

## Setting it up in ScaledPapers

Both checks are ScaledPapers [API checks](/docs/checks/). Here is how the setup above maps to the form.

**What a check measures.** A ScaledPapers check measures **duration**: from the start of the request until the response body has been read, up to the first 64 KB. A clip of one or two words fits well inside that.

**The reachability check.** Method GET. Put your key under **Request headers**. Keep **Expected status code** at 200 and add a **Body contains** or **JSONPath equals** assertion.

**The synthesis canary.** Method POST, a `Content-Type: application/json` header, and the short text in **HTTP body**. Expect 200 and add no body assertion. The key is stored with the check.

**Regions.** The free plan runs each check from one region. Pro runs it from all five: San Francisco, Ashburn, Montreal, Frankfurt, and Singapore.

**Failures in a row.** Under **Advanced**, the **Confirmation period** sets how many failed checks in a row count as down: 1, 2, 3, or 5. The default is 3.

**Slow as a failure.** There is no latency alert. The lever is **Request timeout**: 5, 10, 15, or 30 seconds.

**Latency trend.** On the check page, the **Test duration (avg)** metric follows the time range you pick, back to the start of your plan's history: 14 days on Free, 90 on Pro. The region filter splits it by city, and the samples table shows the duration and status code of each run. The console shows no percentiles. `GET /v1/endpoints/{id}/history` returns `latency_ms` for every sample, so you can compute p50 and p95 yourself.

**Status page.** Link the canary to the "Voice responses" component. Under **When the check is down**, pick **Create an incident automatically**. The default is **Don't create an incident**. If you want a person to approve the incident before it goes public, pick **Ask me to review first** instead.

When your voice agent goes quiet, your users should hear why from you. The free plan includes 10 checks from one region, and Pro checks from all five. See [pricing](/pricing/).
