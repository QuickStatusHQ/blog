# Validation summary

Status
validated

Date: 2026-11-20

## What was checked

Request shapes were compared with the /uptime-rankings/ probe code. Public endpoints were requested live. Every QuickStatus.dev claim was compared with the docs, the console source, and `/pricing/`.

## Result

- **Request shapes**: OpenAI `max_completion_tokens` (GPT-5.4 Nano needs 16 with `reasoning_effort: "none"`), Anthropic `x-api-key` + `anthropic-version: 2023-06-01` + `max_tokens: 1`, Gemini `x-goog-api-key` + `generationConfig.maxOutputTokens: 1`. Source: `data_pipeline/engine/domain/rankings_probes.py`, `data_pipeline/engine/rankings/catalog.json`, `docs/roadmap/2026-11-17-ai-api-uptime-rankings.md` (verified with our keys 2026-11-18).
- **Finish-reason fields**: `$.choices[0].finish_reason`, `$.stop_reason`, `$.candidates[0].finishReason`; reply text is often empty at a tiny limit. Source: same roadmap doc and `rankings_probes.py`.
- **OpenAI-compatible paths**: Groq `/openai/v1/`, Fireworks `/inference/v1/`. Source: `catalog.json`.
- **Unauthenticated reads (2026-11-20, curl)**: `GET api.openai.com/v1/models` 401 "Missing bearer authentication in header"; `GET api.anthropic.com/v1/models` 401 `authentication_error` "x-api-key header is required"; `GET generativelanguage.googleapis.com/v1beta/models` 403 `PERMISSION_DENIED`.
- **Status feeds (2026-11-20, curl)**: `status.openai.com/api/v2/status.json` 200 JSON, indicator `none`; `status.claude.com/api/v2/status.json` 200 JSON, indicator `none`; `status.mistral.ai/api/v2/status.json` 200 JSON, indicator `minor`; `status.x.ai/api/v2/status.json` 403 HTML (bot protection); `aistudio.google.com/status` 200 HTML only.
- **Cost math**: 43,200 runs/month per region at 1 minute; 216,000 from 5 regions; 20 input tokens × 216,000 = 4.32 M (≈ $4.32 at $1/M); output 1–16 tokens × 216,000 at $4/M = $0.86–$13.82. Prices are labelled as an example.
- **Checks**: Up when Status is 200 OK; HTTP request method, headers, body; Body contains is exact and case-sensitive; JSONPath equals compares the string value, so `$.status.indicator` = `none` matches the feeds above. Source: `content/docs/6-checks.md`, `data_pipeline/engine/workers/checkers/http.py`.
- **When to alert**: Mark down after 1/2/3/5 (default 3); Request timeout 5/10/15/30 s; Slow above (ms) on Total time or Time to first byte marks Slow, not down. Source: `content/docs/6-checks.md`.
- **Failure details**: results show the provider's error message, header values replaced with [redacted]; alerts and MCP show the status code alone. Source: `content/docs/7-check-results.md`.
- **Plans**: Free 10 checks, 5 minutes, 1 region; Pro 60 seconds, 5 regions; Scale 30 seconds. Email and Telegram on every plan; Slack, Discord, PagerDuty, Pushover, webhooks on Pro. Source: `public/pricing.md`, `content/docs/6-checks.md`.
- **MCP**: request headers, request body, and HTTP auth stay in the console. Source: `data_pipeline/engine/api/mcp.py`.
- **Rankings**: 8 LLM providers, every 5 minutes from 5 regions. Source: `public/uptime-rankings/llm-apis/`.
- **Latency chart**: the ranges are labelled as rules of thumb, not measurements. The 0.1/1/10 s marks follow Nielsen's response time limits (https://www.nngroup.com/articles/response-times-3-important-limits/, HTTP 200 on 2026-11-20). Band colors are one-hue ink ramps that pass the dataviz ordinal checks in both themes; identity is also carried by the legend, row text, and a table view.

## Not checked

- Authenticated completions were not sent from this session; the request shapes rely on the 2026-11-18 rankings verification.
- Anthropic 529 `overloaded_error`, OpenAI 429 `insufficient_quota`, and 400 low-credit errors are from provider docs, not reproduced.
- The per-use thresholds in the latency chart (code completion, voice, chat, agent step) are editorial judgment, not cited studies.
- Typical finish-reason values (`length`, `max_tokens`, `MAX_TOKENS`) depend on the model.
