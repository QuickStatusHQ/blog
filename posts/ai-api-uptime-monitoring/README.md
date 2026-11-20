---
title: "AI API uptime monitoring: what to probe and when to alert"
seo_title: "AI API uptime monitoring: probe OpenAI, Claude, and Gemini"
description: "How to monitor OpenAI, Anthropic, Gemini, and other AI APIs from outside: the request to send, the status codes that matter, latency limits, and what probes cost."
date: "2026-11-20"
validated: "2026-11-20"
# Target keyword: ai uptime monitoring api (secondary: LLM API uptime monitoring, monitor OpenAI API uptime, check if Claude API is down, AI API latency alerts).
# Competes with getmaxim.ai/articles/ai-outage-monitoring-tools/ (2026-10-13), a gateway-first tools list with no probe requests, status-code handling, latency limits, or probe cost math. This post covers those.
# Request shapes match data_pipeline/engine/domain/rankings_probes.py (the /uptime-rankings/ probes). Facts checked 2026-11-20; see validation/ai-api-uptime-monitoring.md.
# Structure follows docs/marketing/blog-post-structure.md: no product until "Setting it up in QuickStatus.dev".
---

An AI API can fail in more ways than a web page. It can return 500s. It can return 529 or 503 because the provider is overloaded. It can return 429 for your key while everyone else is fine. It can answer 200 in 40 seconds instead of 2. One model can fail while the others work, and one region can fail while the others answer.

Slow is the hardest of these to call, because the same answer time is fine for one feature and an outage for another:

<figure class="lat-figure">
<style>
.lat-figure{--lat-ok:#8f8b80;--lat-mid:#5a574f;--lat-bad:#1e1c17;--lat-label:12.75rem;margin:1.75rem 0;font-family:var(--font-body);}
[data-theme="dark"] .lat-figure{--lat-ok:#6e6b64;--lat-mid:#a3a098;--lat-bad:#e0ded8;}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]) .lat-figure{--lat-ok:#6e6b64;--lat-mid:#a3a098;--lat-bad:#e0ded8;}}
.lat-figure .lat-title{margin:0 0 .35rem;font-weight:600;font-size:1rem;color:var(--color-text);}
.lat-figure .lat-legend{display:flex;flex-wrap:wrap;gap:.35rem 1rem;margin:0 0 1rem;font-size:.85rem;color:var(--color-text-muted);}
.lat-figure .lat-legend span{display:inline-flex;align-items:center;gap:.4rem;}
.lat-figure .lat-sw{display:inline-block;width:.85rem;height:.6rem;border-radius:2px;}
.lat-figure .lat-ok{background:var(--lat-ok);}
.lat-figure .lat-mid{background:var(--lat-mid);}
.lat-figure .lat-bad{background:var(--lat-bad);}
.lat-figure .lat-row{display:grid;grid-template-columns:var(--lat-label) 1fr;align-items:center;gap:.25rem 1rem;padding:.45rem 0;}
.lat-figure .lat-label{line-height:1.3;}
.lat-figure .lat-label b{display:block;font-size:.92rem;font-weight:600;color:var(--color-text);}
.lat-figure .lat-label span{display:block;font-size:.78rem;color:var(--color-text-muted);}
.lat-figure .lat-track{position:relative;height:14px;}
.lat-figure .lat-seg{position:absolute;top:0;bottom:0;border-radius:4px;}
.lat-figure .lat-seg:hover{outline:2px solid var(--color-text);outline-offset:1px;}
.lat-figure .lat-axis{position:relative;height:1.4rem;margin:.35rem 0 0 calc(var(--lat-label) + 1rem);border-top:1px solid var(--color-divider);font-size:.75rem;color:var(--color-text-muted);font-variant-numeric:tabular-nums;}
.lat-figure .lat-axis span{position:absolute;top:.3rem;transform:translateX(-50%);white-space:nowrap;}
.lat-figure .lat-axis span:first-child{transform:none;}
.lat-figure .lat-axis span:last-child{transform:translateX(-100%);}
.lat-figure details{margin-top:.75rem;font-size:.85rem;color:var(--color-text-muted);}
.lat-figure details summary{cursor:pointer;}
.lat-figure details table{margin-top:.5rem;font-size:.85rem;}
.lat-figure figcaption{margin-top:.6rem;color:var(--color-text-muted);font-size:.9rem;line-height:1.5;}
@media (max-width:520px){.lat-figure .lat-row{grid-template-columns:1fr;}.lat-figure .lat-axis{margin-left:0;}.lat-figure .lat-minor{display:none;}}
</style>
<p class="lat-title">How slow is too slow? Time until the user sees the first output</p>
<div class="lat-legend" aria-hidden="true"><span><i class="lat-sw lat-ok"></i>Fine</span><span><i class="lat-sw lat-mid"></i>Users notice</span><span><i class="lat-sw lat-bad"></i>Feels broken</span></div>
<div role="img" aria-label="Rough latency ranges on a log scale from 0.1 to 60 seconds. Code completion: fine under 0.3 s, users notice up to 1 s, broken past 1 s. Voice agent first audio: fine under 0.8 s, notice up to 2 s, broken past 2 s. Chat first token: fine under 1 s, notice up to 10 s, broken past 10 s. Agent step: fine under 10 s, notice up to 30 s, broken past 30 s. Background job: fine until your timeout.">
<div class="lat-row"><div class="lat-label"><b>Code completion</b><span>fine under 0.3 s, broken past 1 s</span></div><div class="lat-track"><span class="lat-seg lat-ok" style="left:0;width:calc(17.17% - 1px)" title="Code completion: fine under 0.3 s"></span><span class="lat-seg lat-mid" style="left:calc(17.17% + 1px);width:calc(18.83% - 2px)" title="Code completion: users notice at 0.3–1 s"></span><span class="lat-seg lat-bad" style="left:calc(36% + 1px);right:0" title="Code completion: feels broken past 1 s"></span></div></div>
<div class="lat-row"><div class="lat-label"><b>Voice agent, first audio</b><span>fine under 0.8 s, broken past 2 s</span></div><div class="lat-track"><span class="lat-seg lat-ok" style="left:0;width:calc(32.51% - 1px)" title="Voice agent: fine under 0.8 s"></span><span class="lat-seg lat-mid" style="left:calc(32.51% + 1px);width:calc(14.32% - 2px)" title="Voice agent: callers notice at 0.8–2 s"></span><span class="lat-seg lat-bad" style="left:calc(46.83% + 1px);right:0" title="Voice agent: feels broken past 2 s"></span></div></div>
<div class="lat-row"><div class="lat-label"><b>Chat, first token</b><span>fine under 1 s, broken past 10 s</span></div><div class="lat-track"><span class="lat-seg lat-ok" style="left:0;width:calc(36% - 1px)" title="Chat: fine under 1 s"></span><span class="lat-seg lat-mid" style="left:calc(36% + 1px);width:calc(35.99% - 2px)" title="Chat: users notice at 1–10 s"></span><span class="lat-seg lat-bad" style="left:calc(71.99% + 1px);right:0" title="Chat: feels broken past 10 s"></span></div></div>
<div class="lat-row"><div class="lat-label"><b>Agent step</b><span>fine under 10 s, broken past 30 s</span></div><div class="lat-track"><span class="lat-seg lat-ok" style="left:0;width:calc(71.99% - 1px)" title="Agent step: fine under 10 s"></span><span class="lat-seg lat-mid" style="left:calc(71.99% + 1px);width:calc(17.17% - 2px)" title="Agent step: users notice at 10–30 s"></span><span class="lat-seg lat-bad" style="left:calc(89.16% + 1px);right:0" title="Agent step: feels broken past 30 s"></span></div></div>
<div class="lat-row"><div class="lat-label"><b>Background job</b><span>fine until your timeout</span></div><div class="lat-track"><span class="lat-seg lat-ok" style="left:0;right:0" title="Background job: fine until your timeout"></span></div></div>
<div class="lat-axis" aria-hidden="true"><span style="left:0">0.1 s</span><span class="lat-minor" style="left:17.17%">0.3 s</span><span style="left:36%">1 s</span><span class="lat-minor" style="left:53.17%">3 s</span><span style="left:71.99%">10 s</span><span style="left:100%">60 s</span></div>
</div>
<details><summary>Show as a table</summary><table><thead><tr><th>Use</th><th>Fine</th><th>Users notice</th><th>Feels broken</th></tr></thead><tbody><tr><td>Code completion</td><td>under 0.3 s</td><td>0.3–1 s</td><td>over 1 s</td></tr><tr><td>Voice agent, first audio</td><td>under 0.8 s</td><td>0.8–2 s</td><td>over 2 s</td></tr><tr><td>Chat, first token</td><td>under 1 s</td><td>1–10 s</td><td>over 10 s</td></tr><tr><td>Agent step</td><td>under 10 s</td><td>10–30 s</td><td>over 30 s</td></tr><tr><td>Background job</td><td>until your timeout</td><td>—</td><td>timeout</td></tr></tbody></table></details>
<figcaption>Rules of thumb for the time until a user sees the first output, not measurements of any provider. The 0.1, 1, and 10 second marks follow Jakob Nielsen's <a href="https://www.nngroup.com/articles/response-times-3-important-limits/">response time limits</a>. Log scale. Measure your own baseline before you set alert limits.</figcaption>
</figure>

The provider's status page catches some of these, usually late. Your own traffic catches the rest, but only when you have traffic. A probe from outside covers the gap: a small real request, on a schedule, from more than one place, that tells you whether your key, your model, and your path to the provider work right now.

This post covers what that probe should send, how to read its answer, when to alert, and what it costs.

## Three signals, three different questions

| Signal | What it answers | Cost |
| --- | --- | --- |
| The provider's status feed | Has the provider declared an incident? | Nothing |
| An authenticated read, such as `GET /v1/models` | Is the API reachable, and does my key work? | No tokens |
| A tiny request to the model you ship | Does my model answer, and how fast? | A few tokens per run |

<figure class="sig-figure">
<style>
.sig-figure{margin:1.75rem 0;}
.sig-figure svg{display:block;width:100%;height:auto;max-width:600px;font-family:var(--font-body);}
.sig-figure .sig-box{fill:var(--color-surface-offset);stroke:var(--color-border);stroke-width:1;}
.sig-figure .sig-frame{fill:none;stroke:var(--color-border);stroke-width:1;stroke-dasharray:4 4;}
.sig-figure .sig-title{fill:var(--color-text);font-size:15px;font-weight:600;}
.sig-figure .sig-sub{fill:var(--color-text-muted);font-size:12.5px;}
.sig-figure .sig-code{fill:var(--color-text-muted);font-size:12px;font-family:var(--font-mono);}
.sig-figure .sig-line{stroke:var(--color-text-muted);stroke-width:2;fill:none;}
.sig-figure .sig-dot{fill:var(--color-text-muted);}
.sig-figure .sig-line--key{stroke:var(--color-success);stroke-width:2.5;}
.sig-figure .sig-dot--key{fill:var(--color-success);}
.sig-figure .sig-title--key{fill:var(--color-success);}
.sig-figure .sig-later{stroke:var(--color-text-faint);stroke-width:1.5;stroke-dasharray:3 4;fill:none;}
.sig-figure .sig-tall{display:none;max-width:400px;}
@media (max-width:520px){.sig-figure .sig-wide{display:none;}.sig-figure .sig-tall{display:block;}}
.sig-figure figcaption{margin-top:.6rem;color:var(--color-text-muted);font-size:.9rem;line-height:1.5;}
</style>
<svg class="sig-wide" viewBox="0 0 560 320" role="img" aria-labelledby="sig-title sig-desc">
<title id="sig-title">How far each AI API check reaches</title>
<desc id="sig-desc">The status feed check reads the provider's status page, which staff update after an incident is confirmed. The cheap read, GET /v1/models, reaches the API front door and proves the network, TLS, and your key. Only the tiny model request goes through the front door to the model and proves it can answer.</desc>
<rect class="sig-box" x="330" y="14" width="220" height="54" rx="8"/>
<text class="sig-title" x="440" y="37" text-anchor="middle">Status page</text>
<text class="sig-sub" x="440" y="56" text-anchor="middle">what the provider declared</text>
<rect class="sig-frame" x="232" y="112" width="322" height="202" rx="10"/>
<text class="sig-sub" x="246" y="131">Provider API</text>
<rect class="sig-box" x="246" y="142" width="134" height="160" rx="8"/>
<text class="sig-title" x="313" y="165" text-anchor="middle">Front door</text>
<text class="sig-sub" x="313" y="184" text-anchor="middle">network, TLS,</text>
<text class="sig-sub" x="313" y="200" text-anchor="middle">your key</text>
<rect class="sig-box" x="404" y="142" width="136" height="160" rx="8"/>
<text class="sig-title" x="472" y="165" text-anchor="middle">Model</text>
<text class="sig-sub" x="472" y="184" text-anchor="middle">capacity to</text>
<text class="sig-sub" x="472" y="200" text-anchor="middle">answer</text>
<path class="sig-later" d="M472 142 V70"/>
<text class="sig-sub" x="480" y="96">posted later</text>
<text class="sig-title" x="0" y="36">Status feed</text>
<text class="sig-code" x="0" y="54">status.json</text>
<path class="sig-line" d="M150 41 H322"/>
<circle class="sig-dot" cx="324" cy="41" r="4.5"/>
<text class="sig-title" x="0" y="218">Cheap read</text>
<text class="sig-code" x="0" y="236">GET /v1/models</text>
<path class="sig-line" d="M150 224 H313"/>
<circle class="sig-dot" cx="313" cy="224" r="4.5"/>
<text class="sig-title sig-title--key" x="0" y="272">Model request</text>
<text class="sig-code" x="0" y="290">POST, tiny limit</text>
<path class="sig-line sig-line--key" d="M150 278 H472"/>
<circle class="sig-dot sig-dot--key" cx="472" cy="278" r="5"/>
</svg>
<svg class="sig-tall" viewBox="0 0 340 460" role="img" aria-labelledby="sig-title-t sig-desc-t">
<title id="sig-title-t">How far each AI API check reaches</title>
<desc id="sig-desc-t">The cheap read, GET /v1/models, stops at the API front door and proves the network, TLS, and your key. The tiny model request goes through the front door to the model and proves it can answer. The status feed check reads the provider's status page, which is posted after an incident is confirmed.</desc>
<text class="sig-title" x="24" y="24">Cheap read</text>
<text class="sig-code" x="24" y="42">GET /v1/models</text>
<text class="sig-title sig-title--key" x="190" y="24">Model request</text>
<text class="sig-code" x="190" y="42">POST, tiny limit</text>
<rect class="sig-frame" x="10" y="64" width="320" height="256" rx="10"/>
<text class="sig-sub" x="24" y="86">Provider API</text>
<rect class="sig-box" x="24" y="98" width="292" height="92" rx="8"/>
<text class="sig-title" x="170" y="136" text-anchor="middle">Front door</text>
<text class="sig-sub" x="170" y="156" text-anchor="middle">network, TLS, your key</text>
<rect class="sig-box" x="24" y="208" width="292" height="98" rx="8"/>
<text class="sig-title" x="170" y="246" text-anchor="middle">Model</text>
<text class="sig-sub" x="170" y="266" text-anchor="middle">capacity to answer</text>
<path class="sig-line" d="M70 52 V164"/>
<circle class="sig-dot" cx="70" cy="166" r="4.5"/>
<path class="sig-line sig-line--key" d="M250 52 V280"/>
<circle class="sig-dot sig-dot--key" cx="250" cy="282" r="5"/>
<path class="sig-later" d="M170 306 V350"/>
<text class="sig-sub" x="178" y="336">posted later</text>
<rect class="sig-box" x="24" y="350" width="292" height="54" rx="8"/>
<text class="sig-title" x="170" y="373" text-anchor="middle">Status page</text>
<text class="sig-sub" x="170" y="393" text-anchor="middle">what the provider declared</text>
<path class="sig-line" d="M70 444 V410"/>
<circle class="sig-dot" cx="70" cy="408" r="4.5"/>
<text class="sig-title" x="84" y="436">Status feed</text>
<text class="sig-code" x="84" y="452">status.json</text>
</svg>
<figcaption>Each check reaches one layer further. The status page is updated by people after an incident is confirmed, and the cheap read stops at the front door. Only the model request proves the model can answer, so it is the one to alert on.</figcaption>
</figure>

Run all three. The status feed gives you context and wording for your own status page. The read is cheap enough to run often. The model request is the only one that proves the model works, so it is the one that should page someone.

If your app already logs every model call, error rates from that traffic are the best signal during busy hours. Keep the probe anyway: it runs at 3 a.m. when nobody is calling the API, it gives you a clean latency baseline with the same request every time, and it still works when the failure is in your own code path.

## The request to send

Send the smallest request your app actually makes, to the model your app actually uses, with a key reserved for monitoring. Keep the prompt short and the output limit as low as the model accepts.

**OpenAI**

```
curl -s https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_MONITOR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"$MODEL"'","messages":[{"role":"user","content":"Reply with: ok"}],"max_completion_tokens":16,"reasoning_effort":"none"}'
```

Newer OpenAI models reject `max_tokens` and need `max_completion_tokens`. Reasoning models count their thinking against that limit, and some reject a limit of 1, so leave a little room and set the lowest `reasoning_effort` the model accepts. Drop `reasoning_effort` for models that do not reason.

**Anthropic**

```
curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_MONITOR_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"'"$MODEL"'","max_tokens":1,"messages":[{"role":"user","content":"Reply with: ok"}]}'
```

**Google Gemini**

```
curl -s "https://generativelanguage.googleapis.com/v1beta/models/$MODEL:generateContent" \
  -H "x-goog-api-key: $GEMINI_MONITOR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"Reply with: ok"}]}],"generationConfig":{"maxOutputTokens":1}}'
```

Put the Gemini key in the header, not in a `?key=` query parameter. URLs end up in logs and alert messages, where a header does not.

**OpenAI-compatible APIs.** Groq, Mistral, xAI, DeepSeek, Together, Fireworks, and OpenRouter accept the OpenAI request shape at their own `/chat/completions` URL with `Authorization: Bearer`. Check the exact path in each provider's docs: some put it under `/openai/v1/` or `/inference/v1/`.

### Check the finish reason, not the text

With a tiny output limit, the reply text is often empty, especially from reasoning models. An assertion like "the response contains ok" fails on a model that is working fine. Assert that the finish-reason field is present instead:

| Provider | Field | Typical value with a tiny limit |
| --- | --- | --- |
| OpenAI and compatible | `$.choices[0].finish_reason` | `length` or `stop` |
| Anthropic | `$.stop_reason` | `max_tokens` or `end_turn` |
| Gemini | `$.candidates[0].finishReason` | `MAX_TOKENS` or `STOP` |

A 200 with that field means the model ran and produced output. A 200 without it is worth alerting on: [a 200 can still be down](/blog/a-200-that-is-still-down/).

### The cheap read

For a check that runs often, call the model list with your key. It costs no tokens.

```
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" \
  https://api.anthropic.com/v1/models \
  -H "x-api-key: $ANTHROPIC_MONITOR_KEY" -H "anthropic-version: 2023-06-01"
```

The same read works at `https://api.openai.com/v1/models` with `Authorization: Bearer` and at `https://generativelanguage.googleapis.com/v1beta/models` with `x-goog-api-key`. Without a key, OpenAI and Anthropic answer 401 and Gemini answers 403, so a monitor that treats any response below 500 as up will report all three as up with no key at all. Expect exactly 200.

This proves the network path, TLS, the API front door, and your key. It does not prove that any model can serve a request. Model capacity fails separately, and that is the common failure.

## What the status code means

| Answer | What it usually means | Alert? |
| --- | --- | --- |
| 200 with a finish reason | Up | No |
| 401 or 403 | Your key was revoked, expired, or lost access to the model | Yes, to you. Users see an outage even though the provider is fine |
| 404 | The model name is wrong or was retired | Yes. Model retirements break apps on a known date |
| 429 | Your rate limit or quota. OpenAI also answers 429 when the account is out of credit | Yes, if it repeats. Check billing before blaming the provider |
| 400 | A bad request body, or on some providers a billing problem such as a low credit balance | Yes. Read the error message |
| 500, 502, 503, 504 | A provider-side failure | Yes, after confirmation |
| 529 (Anthropic) | The API is overloaded | Yes, after confirmation. Retries often succeed within minutes |
| Timeout | The provider or the path to it is too slow to answer | Yes, after confirmation |

Read the error body as well as the code. Providers put the reason in JSON, for example `{"type":"error","error":{"type":"authentication_error",...}}` from Anthropic, and that text is what tells a billing problem apart from an outage.

## Read the status feed too

Most AI providers publish a machine-readable status summary. These answered with JSON on 2026-11-20:

- `https://status.openai.com/api/v2/status.json`
- `https://status.claude.com/api/v2/status.json` (Anthropic)
- `https://status.mistral.ai/api/v2/status.json`

Each returns `"indicator"`: `none` when nothing is declared, then `minor`, `major`, or `critical`. Some feeds are HTML only, such as Gemini's at `aistudio.google.com/status`, and some sit behind bot protection that blocks plain requests.

A status feed is a lagging signal. A provider posts after it has confirmed a problem, a "partial outage" banner may or may not cover your model, and a quota or key problem never shows up there. Use it to explain an alert, not to raise one.

## Latency: decide what slow means

A model that answers in 40 seconds is down for a chat UI and fine for a nightly batch. The chart at the top gives rough ranges per use. For alerts, pick the limit from your own numbers:

1. Run the probe for a week and note the typical time and the slowest 5% from each region.
2. Set the slow limit at about two to three times that slow-5% figure. Lower makes noise, higher hides real slowdowns.
3. Set the timeout well above the slow limit, usually 15 or 30 seconds for a model call, so a slow answer reads as slow, not as down.
4. Alert on slow separately from down. Slow usually means degraded capacity, and it often comes before a full outage.

Measure time to first byte as well as total time. For a non-streaming request with a tiny output the two are close. For streaming apps, time to first token is what users feel. A probe with `"stream": true` and a time-to-first-byte limit approximates it, but it cannot see a stream that stalls halfway through a long answer. That needs measurement in your own app.

The tiny probe also measures the best case. Long prompts, long context, and long outputs take longer, so your app's real latency sits above the probe. Compare like with like: the probe against its own history, not against your production p95.

## What probes cost

Requests per month, per region:

| Interval | Requests per region | From 5 regions |
| --- | --- | --- |
| 30 seconds | 86,400 | 432,000 |
| 1 minute | 43,200 | 216,000 |
| 5 minutes | 8,640 | 43,200 |

A request like the ones above uses about 10 to 20 input tokens and 1 to 16 output tokens. At an example price of $1 per million input tokens and $4 per million output tokens, every minute from 5 regions is about 4.3 million input tokens a month, roughly $4, plus $1 to $14 of output depending on the limit. Every 5 minutes from 5 regions is a fifth of that. Small models cost a fraction of it. Check the current price of the model you probe and do the same arithmetic.

Two rules keep the bill and the blast radius small:

- **Use a separate key with a hard spending limit.** A probe should never be able to run up a large bill, and a revoked monitoring key should not take down production.
- **Probe the model you ship, not a cheaper one.** A cheap model being up tells you little about the model your users hit. If cost matters, probe the cheap read often and the real model every 5 minutes.

## Confirm before you page anyone

Model APIs time out now and then, and a single failed probe is usually noise. [One failed check is noise](/blog/one-failed-check-is-noise/) covers this in detail. For AI APIs:

- **Wait for a pattern.** Alert after 2 or 3 failed runs in a row.
- **Check from more than one region.** If a provider fails from Frankfurt and works from Virginia, that is a regional or routing problem, and your EU users need to hear about it. If every region fails, it is the provider. [Why we ping from different regions](/blog/why-we-ping-from-different-regions/) explains what each pattern means.
- **Alert per provider and per model.** One alert for "AI is down" hides which fallback to switch to.

If you route between providers, through a gateway or your own fallback code, probe each provider directly and probe your own endpoint too. The direct probes tell you which provider failed. The probe on your endpoint tells you whether the fallback worked.

## What an outside probe does not cover

- **Answer quality.** A model that answers 200 with wrong or empty output for real prompts looks up. Evaluations and output checks belong in your app.
- **Your real latency.** The probe measures a tiny request. Long prompts and long outputs need measurement in your own traffic.
- **Streams that stall.** A probe sees the first bytes. It does not see a stream that stops halfway through.
- **Per-customer problems.** If each customer brings their own key, one probe with your key cannot see their quota.
- **Automatic failover.** A probe tells you a provider failed. Switching traffic is your code's or your gateway's job.

## A starting list

1. A tiny request to each model you ship, with a monitoring key, asserting status 200 and the finish-reason field.
2. A `GET /v1/models` read per provider, expecting exactly 200, on a shorter interval if you want earlier warning of key problems.
3. A check on each provider's status feed, used to explain alerts.
4. A timeout of 15 to 30 seconds, and a separate slow limit at two to three times your measured slow-5% time.
5. Alerts after 2 or 3 failures in a row, from at least two regions, per provider and per model.
6. A monitoring key with a spending limit, and the cost worked out before you shorten the interval.

The status page tells you what the provider admits. Your own probe tells you whether your app can call the model right now.

## Setting it up in QuickStatus.dev

Everything above works with any monitor that can send a POST with secret headers and check the body. In QuickStatus.dev, each probe is one HTTP check.

**The model request.** Open **Checks**, click **Add check**, and pick **Website or API**. Enter the provider URL and set **Up when** to **Status is 200 OK**. Under **HTTP request**, set the method to POST, add the auth header and `Content-Type: application/json` under **Request headers**, and paste the request body. Under **Assertions**, add **Body contains** with the finish-reason field name: `finish_reason`, `stop_reason`, or `finishReason`. It is an exact, case-sensitive text match. [Monitor the OpenAI API](/monitor-openai-api/) walks through the same steps for OpenAI.

**The cheap read and the status feed.** The `/v1/models` read is the same check with method GET and no body. For a status feed, point an HTTP check at its `status.json` URL and add **JSONPath equals** with `$.status.indicator` equal to `none`. The check then goes down when the provider declares an incident, and you can link it to its own status page component.

**When to alert.** Under **When to alert**, set **Mark down after** to 2 or 3 and **Request timeout** to 15 or 30 seconds. Under **Slow responses**, set **Slow above (ms)** and measure on **Total time** or **Time to first byte**. A slow run marks the check **Slow**, not down, sends an alert, and leaves uptime alone.

**Regions and intervals.** Free checks every 5 minutes from 1 region. Pro checks every 60 seconds from up to five regions: San Francisco, Ashburn, Montreal, Frankfurt, and Singapore. Scale checks every 30 seconds. Every run stores its duration and region, so the check page shows whether a slowdown is everywhere or in one place.

**What failed.** When a run fails, the check's results show what the provider answered, such as `authentication_error` or an overloaded message, with header values replaced by [redacted]. Alerts carry the status code. Email and Telegram alerts are on every plan. Slack, Discord, PagerDuty, Pushover, and webhook alerts are Pro.

**Status page.** Link each check to a component, for example *Chat (OpenAI)* or *Summaries (Anthropic)*, so a provider outage shows as one feature degraded rather than everything down. Under **When the check is down**, choose **Create an incident automatically** or **Ask me to review first**.

**How the providers compare.** [LLM API uptime rankings](/uptime-rankings/llm-apis/) runs this probe, a tiny request with a finish-reason check, against OpenAI, Anthropic, Gemini, xAI, Mistral, DeepSeek, Cohere, and Moonshot every 5 minutes from all five regions, and publishes weekly uptime.

**Limits.** Checks only reach public URLs, so a model server inside your VPC needs a public health route. A check measures one request: it does not follow a stream to the end, score answers, or fail traffic over to another provider. Auth headers and request bodies are set in the console. The [uptime monitoring MCP server](/mcp-uptime-monitoring/) can create, run, and link these checks from Claude Code, Cursor, or Codex, but it leaves secrets in the console.

When the model API your product depends on fails, your users should hear it from you before they open a ticket. Start free with 10 checks and a status page, no card. See [pricing](/pricing/).
