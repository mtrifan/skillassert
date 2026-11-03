# 02 - LLM question generation: cost and capabilities

- Date read: 2026-10-29. All prices are USD list prices per 1M tokens, excluding tax and card fees.
- Scope: batch generation of multiple-choice questions for skillassert.com, plus an optional verification pass. Facts and arithmetic only. No recommendation, no winner.
- Sources: Anthropic ids, prices, structured output, caching and batch come from the `claude-api` skill (its model table is dated 2026-10-06) and were cross-checked against platform.claude.com pages read today. OpenAI and Google facts come from official docs fetched today as raw markdown or HTML. Source keys (A1, O1, G1, X1) are listed at the end.
- One discrepancy: the skill lists Sonnet 5.5 cache reads at $0.20 per M; the live pricing page says $0.10 (0.05x). The live value is used below.
- Budget in USD: EUR 20 per month is about USD 22.27 at the ECB reference rate of 1.1133 USD per EUR on 2026-10-28 [X1].

## 1. Summary table

| Provider / model id | $/M in | $/M out | Structured output | Batch | Caching (read / write, min prefix) | Free credits | EU option | Src |
|---|---|---|---|---|---|---|---|---|
| `claude-haiku-5-5` (smallest) | 0.10 | 0.50 (0.50 / 2.50 if prompt over 100K) | `output_config.format` json_schema | -50%: 0.05 / 0.25 | read 0.01, 5m write 0.125; min 512 | "small amount", size unstated | none first-party (`inference_geo` us or global) | A1 A2 |
| `claude-sonnet-5-5` (mid) | 2.00 | 10.00 | same | -50%: 1 / 5 | read 0.10, write 2.50; min 512 | same | same | A1 A2 |
| `claude-opus-5-5` | 4.00 | 20.00 | same | -50%: 2 / 10 | read 0.20, write 5.00; min 512 | same | same | A1 A2 |
| `gpt-6-luna` (smallest) | 0.10 | 0.50 | `text.format` json_schema strict | -50%: 0.05 / 0.25 | read 0.01, 30m write 0.125; min 1,024 | no new-account credit; Pro/Business subscriber credits and opt-in data-sharing daily tokens (see Q6) | `eu.api.openai.com`, +10%, sales-gated | O1 O2 |
| `gpt-6.1-terra` | 1.00 | 5.00 | same | -50%: 0.50 / 2.50 | read 0.05, write 1.25 | same | same | O1 O2 |
| `gpt-6.1-sol` (standard) | 2.00 | 10.00 | same | -50%: 1 / 5 | read 0.10, write 2.50 | same | same | O1 O2 |
| `gpt-6-astra` (top) | 10.00 | 50.00 | same | -50%: 5 / 25 | read 1.00, write 12.50 | same | same | O1 O3 |
| `gpt-5.4-mini` / `gpt-5.4-nano` (older small) | 0.75 / 0.20 | 4.50 / 1.25 | same | -50% | read 0.075 / 0.02, no write fee | same | not checked | O1 |
| `gemini-3.1-flash-lite` | 0.25 | 1.50 | `response_format` json (Interactions) or `response_mime_type` + `response_schema` | -50%: 0.125 / 0.75 | implicit, read 0.025; min not listed | free tier, no billing account | none on Developer API; Cloud EU +10% | G1 G3 G4 |
| `gemini-3.5-flash-lite` | 0.30 | 2.50 | same | -50%: 0.15 / 1.25 | implicit, read 0.03; min not listed | free tier, no billing account | same | G1 G3 G4 |
| `gemini-3.6-flash` | 0.75 (1.50 from 2027-01-01) | 3.75 (7.50) | same | -50%, not on free tier | implicit, read 0.075; min 4,096 | free tier, no billing account | same | G1 G5 |
| `gemini-3.8-flash` (Flash) | 0.75 (1.50 from 2027-01-01) | 3.75 (7.50) | same | -50%: 0.375 / 1.875, not on free tier | implicit, read 0.075; min 4,096 | free tier, no billing account | same | G1 G5 |
| `gemini-3.1-pro-preview` (Pro, preview) | 2.00 (4.00 over 200K) | 12.00 (18.00 over 200K) | same | -50%: 1 / 6 | read 0.20; min 4,096 | no free tier | same | G1 G5 |
| `gemini-4-argon` (flagship) | 2.00 (4.00 from 2026-12-01) | 10.00 (20.00) | same | -50%: 1 / 5 | implicit only, read 0.10; min 4,096 | no free tier | same | G1 G5 |

Notes: `claude-fable-5-1` is $10 / $50 (batch 5 / 25) [A2]; older `claude-haiku-4-5` is $1 / $5 [A2]. OpenAI shows the same cached and batch structure for every gpt-6 row [O1]. Gemini output prices include thinking tokens [G1]. Anthropic Batch combines with caching discounts [A2 A5]. Gemini batch and cache prices move on the same dates as the standard prices.

## 2. Details behind the table

### Q2. Structured output (JSON Schema constrained decoding)
- Anthropic [A1 A4]: `output_config: { format: { type: "json_schema", schema } }`; TypeScript SDK `client.messages.parse()` with `zodOutputFormat()`. Works with Batches. Limits: `additionalProperties: false` on every object; no recursive schemas; no minimum/maximum/multipleOf; no minLength/maxLength; array `minItems` only 0 or 1; 24 optional and 16 union-typed parameters per request; 20 strict tools. SDK helpers strip unsupported keywords and validate client-side. New schema pays a one-time compile, cached 24 h. A refusal or max_tokens stop can leave output off-schema.
- OpenAI [O4]: Responses `text: { format: { type: "json_schema", strict: true, schema } }`; Chat Completions `response_format`. Works with Batch. Limits: `additionalProperties: false`, every property in `required` (optional = union with null); 5,000 object properties, 10 nesting levels, 120,000 chars of names/enums, 1,000 enum values. Supports minItems/maxItems, minimum/maximum, pattern, format.
- Google [G3]: Interactions API `response_format: { type: "text", mime_type: "application/json", schema }`. generateContent ("remains fully supported") uses `response_mime_type` + `response_schema`. Subset of JSON Schema; minItems/maxItems, enum, minimum/maximum work; minLength/maxLength/uniqueItems are ignored. Oversized schemas return 400. No numeric limits published.

### Q3. Batch
- Anthropic [A5]: 50% off all tokens; up to 100,000 requests or 256 MB per batch; most batches end within 1 hour, maximum 24 hours; results kept 29 days; cache hits inside a batch are best-effort. Message Batches is not on Bedrock, Vertex or Foundry [A1].
- OpenAI [O5]: 50% off; `completion_window` only `24h`; up to 50,000 requests and 200 MB per file; 2,000 batches per hour; separate rate-limit pool with a queued-token cap per model.
- Google [G4 G6]: 50% off; target turnaround 24 h, often quicker; inline requests or JSONL up to 2 GB; 100 concurrent batch requests; generateContent only, not Interactions.

### Q4. Prompt caching, and the effect on a repeated 1,500-token system prompt
- Anthropic [A1 A2 A6]: opt-in `cache_control`; 5 min default, 1 h option; write 1.25x (2x for 1 h), read 0.1x (0.05x on Sonnet 5.5 and Opus 5.5); minimum 512 tokens, so 1,500 tokens qualifies.
- OpenAI [O2 O6]: on by default; GPT-5.6 and later bill a write at 1.25x (30 min) or 2.5x (12 h), read 0.1x (0.05x on Sol and Terra); minimum 1,024 visible tokens, so 1,500 qualifies.
- Google [G5]: implicit, no setup, no write fee, read 10% of input; minimum 4,096 tokens on 3.8 Flash, 3.6 Flash, 3.1 Pro and Argon, so 1,500 does NOT qualify; Flash-Lite minimum is not listed. Explicit caching only via generateContent on Gemini 3 and 2.5, with hourly storage fees.
- Effect for 500 requests sharing a 1,500-token prefix (1 write, 499 reads inside the cache lifetime), input cost only: Haiku 5.5 or gpt-6-luna $0.075 falls to $0.0077; gpt-6.1-terra $0.75 to $0.039; Sonnet 5.5 or gpt-6.1-sol $1.50 to $0.079; Opus 5.5 $3.00 to $0.157; Gemini 3.8 Flash, 3.6 Flash, 3.1 Pro, Argon no change; Flash-Lite unknown.

### Q5. Rate limits for a brand-new, low-spend account
| Provider | Entry tier and how to reach it | RPM | Token limits | Batch queue |
|---|---|---|---|---|
| Anthropic [A3] | Start tier (new orgs "may start in the Evaluation tier" with lower, unpublished limits) | 1,000 for Haiku 5.5, Sonnet 5.5, Opus 5.5 | 2,000,000 input / 400,000 output per min; cached reads do not count | 200,000 requests; 100,000 per batch |
| OpenAI [O7 O2] | Build: $5 total credit purchases | 5,000 | luna 2,000,000 TPM; terra, sol, astra 1,000,000 TPM | luna 20,000,000 tokens; terra 3,000,000 |
| Google [G6 G7] | Tier 1: billing account linked, prepay of at least $5; Free tier needs no billing | not published in docs (AI Studio only) | not published | 3.5 Flash-Lite 10M; 3.8 Flash 3M; Argon 5M enqueued tokens |

- 500 questions at the stated assumptions are 500 requests, 0.75M to 1.5M input tokens and 0.30M to 0.45M output tokens. At the high end that is 450,000 output tokens against Anthropic's 400,000 per minute (about 1.1 minutes of quota) and 1.95M total tokens against luna's 2,000,000 TPM (about 1 minute, if TPM counts input plus output, which the guide does not spell out); terra, sol and astra at 1,000,000 TPM need about 2 minutes. Wall-clock time is then set by per-request latency, which was not measured.
- Ramp limits: Anthropic warns of acceleration limits on sharp increases [A3]; OpenAI returns 429 `slow_down` on rapid ramps and advises at most +50% per 15 minutes above 1M TPM [O7].
- Hard stops: Anthropic Start cap $500 per month (HTTP 429 at the cap, HTTP 400 at a limit you set yourself) [A3]; OpenAI Build usage limit $500 per month and a hard spend limit returns 429 [O7]; Gemini Prepay at $0 balance returns HTTP 402 with about 10 minutes of lag [G7].

### Q6. Free credits, free tiers, card requirement
- Anthropic [A2]: "New users receive a small amount of free credits to test the API." No amount, expiry or card rule is documented. No ongoing free tier. Standard accounts pay by credit card.
- OpenAI [O7]: tier table lists Free (allowed geography, $100/month usage limit) with no per-model limits shown; Build needs $5 in credit purchases. No new-account credit found. Two other programs exist and are detailed in the next two bullets.
- OpenAI, ChatGPT subscription versus API (official pages read 2026-10-29: O10, O11, O12). API usage is always billed separately per token; a plan can only add credits:
  - Separate billing: "API token prices are separate from subscription usage; don't use them to estimate included tasks" [O11]. With an API key, "OpenAI bills API key usage through your OpenAI Platform account at standard API rates" and "Codex uses standard API pricing instead of included ChatGPT plan credits" [O12]. Plus includes Codex when signed in with ChatGPT, and Plus or Pro users can buy extra ChatGPT credits to extend that usage; running Codex with an API key is charged at standard API rates instead [O11]. None of this is Responses or Chat Completions API access.
  - Plus (also Go and Free): no API credits; the credits page names them as ineligible, and Enterprise too [O10].
  - Pro and Business: can claim monthly API credits "while these benefits are available" [O10]. Pro gets $100, $200 or $500 per month by plan tier. Business gets $20 per paid Standard seat or $100 per paid Premium seat, pooled and capped at $500 per workspace per month. No plan called "Team" is named in these docs; the Business sign-up link is chatgpt.com/team-sign-up [O11].
  - Conditions [O10]: paid plan at least 7 days old (trials and complimentary plans do not qualify); you own both the subscription and the API organization, with the same email; credits go only to that organization, are used before purchased credits, expire at the end of each billing cycle and do not roll over. With no credits left and none purchased, requests fail with HTTP 429 and error code `credit_balance_exhausted`. No payment method is needed while credits are active.
  - Coverage [O10]: Responses, Agents, Decisions, Realtime and Images APIs and built-in tools are named; Chat Completions is not named ("Other APIs are available through API Platform"). Not valid for Codex through an API key, and the credits do not raise signed-in Codex limits.
- OpenAI, complimentary daily tokens for shared data: a program exists for "some organizations" [O13]. The official help page returned HTTP 403 to direct fetch, so it was read from the Internet Archive snapshot of 2026-10-17 (page marked "Updated: 9 days ago" at that time):
  - Conditions: an organization owner opts in to share inputs and outputs (off by default; per project or organization; not available to Enterprise or Zero Data Retention accounts); eligibility shows on the data-sharing settings page as "You're eligible for free daily usage on traffic shared with OpenAI", and without that text "you are not eligible for the complimentary tokens at this time"; a positive account balance is required; only traffic on opted-in projects counts; fine-tuned models, fine-tuning training, evals and tool use are excluded; opt-out is possible at any time. Shared data is used "to help identify usage patterns, measure model quality, and inform future evaluation and training of models", and the page says not to share sensitive, confidential or proprietary information.
  - Amounts per day: Build tier 250,000 tokens for one model group and 2,500,000 for another; Launch and Grow 1,000,000 and 10,000,000. The first group lists gpt-6-luna, gpt-6.1-sol, gpt-6-sol, gpt-6-astra and gpt-5.6-sol; the second lists gpt-5.6-terra, gpt-5.6-luna, gpt-5.4-mini, gpt-5.4-nano, gpt-5-mini and gpt-5-nano. gpt-6.1-terra is not listed in either.
  - Mechanics: the quota is shared inside a group and resets at 00:00 UTC; a request that would push the daily total over the quota is billed in full at normal rates; the page says "OpenAI will give 30 days notice before terminating this program".
  - Scale at the brief's sizes: 250,000 tokens per day on gpt-6-luna is worth $0.025 (all input at $0.10) to $0.125 (all output at $0.50), about 64 to 119 questions per day (250,000 / 3,900 and 250,000 / 2,100 tokens). 2,500,000 tokens per day on gpt-5.6-luna ($0.20 / $1.20) is worth $0.50 to $3.00 and exceeds the 1.05M to 1.95M tokens of a 500-question launch.
- Google [G1 G7 G8]: Free tier needs no billing account (a key is created by default); input and output are free on 3.8 Flash, 3.6 Flash and both Flash-Lite models; Argon and 3.1 Pro are not available free. Paid tier needs a linked billing account with a payment method and either a Prepay of at least $5 or, where offered, Postpay; prepaid credits expire after 12 months.
- Gemini free-tier data caveats [G1 G8]: the pricing page marks free-tier content as "used to improve our products: Yes", and the terms say human reviewers may read it. The terms also state: "If you're in the European Economic Area, Switzerland, or the United Kingdom, the terms under 'How Google uses Your Data' in 'Paid Services' apply to all Services, including Google AI Studio and unpaid quota in the Gemini API". And: "You may use only Paid Services when making API Clients available to users in the European Economic Area, Switzerland, or the United Kingdom." (terms effective 2026-03-23).

### Q7. EU data residency and endpoints
- Anthropic [A7 A2]: first-party `inference_geo` accepts only `us` (1.1x on all token prices) or `global` (default); workspace geo only `us`. No EU value. On Bedrock and Google Cloud (Google managed) the region follows the endpoint, and regional or multi-region endpoints carry a 10% premium over global. EU model availability per region was not checked.
- OpenAI [O1 O2 O8]: `eu.api.openai.com` gives EU processing and storage for eligible models, Standard, Fast, Flex and Batch, at +10% for models released on or after 2026-03-05. Docs say to contact sales for eligibility; non-US regions need approval for abuse-monitoring controls and a Modified Retention amendment. API data is not used for training by default; abuse logs are kept up to 30 days.
- Google [G8 G10]: the Gemini Developer API documents no region choice, and its terms say paid-service data "may be stored transiently or cached in any country in which Google or its agents maintain facilities". The Google Cloud Gemini platform offers an EU multi-region endpoint and European regional endpoints (table marks EU multi-region supported for Argon, 3.8 Flash, 3.6 Flash, 3.5 Flash-Lite, 3.1 Flash-Lite). Cloud pricing lists "Non-global" about 10% above "Global" (3.8 Flash: input $0.825 vs $0.75, output $4.125 vs $3.75 through 2026-12-31); EU endpoints are not global endpoints per the residency page. The Interactions API on that platform supports only the global endpoint during Preview.

## 3. Worked cost examples

Assumptions (from the brief): one question per request; draft = 1,500 (low) to 3,000 (high) input tokens and 600 (low) to 900 (high) output tokens; verification = 1,000 input and 150 output tokens; hidden reasoning tokens are assumed to be inside these counts.

Formula: `cost = Q x (T_in x P_in + T_out x P_out) / 1,000,000`, and batch = 0.5 x standard. Prices are the table values (use the Gemini post-promotion values where marked).

Example: Haiku 5.5, low = (1,500 x 0.10 + 600 x 0.50) / 1e6 = $0.00045 per question, x 500 = $0.225. Verify on Sonnet 5.5 = (1,000 x 2.00 + 150 x 10.00) / 1e6 = $0.0035 per question, x 500 = $1.75; total $1.975.

Each cell is low-high (rounded half up). "Per month" is the 100-per-week column times 52/12 = 4.33 (433 questions).

| Pipeline | 500 q standard | 500 q batch | 100 q/week standard | 100 q/week batch | per month (x4.33, std) |
|---|---|---|---|---|---|
| Anthropic: claude-haiku-5-5 only | $0.225-0.375 | $0.113-0.188 | $0.045-0.075 | $0.023-0.038 | $0.195-0.325 |
| Anthropic: haiku-5-5 draft + claude-sonnet-5-5 verify | $1.98-2.13 | $0.988-1.06 | $0.395-0.425 | $0.198-0.213 | $1.71-1.84 |
| Anthropic: haiku-5-5 draft + claude-opus-5-5 verify | $3.73-3.88 | $1.86-1.94 | $0.745-0.775 | $0.373-0.388 | $3.23-3.36 |
| Anthropic: claude-sonnet-5-5 only (mid, reference) | $4.50-7.50 | $2.25-3.75 | $0.900-1.50 | $0.450-0.750 | $3.90-6.50 |
| OpenAI: gpt-6-luna only | $0.225-0.375 | $0.113-0.188 | $0.045-0.075 | $0.023-0.038 | $0.195-0.325 |
| OpenAI: luna draft + gpt-6.1-sol verify | $1.98-2.13 | $0.988-1.06 | $0.395-0.425 | $0.198-0.213 | $1.71-1.84 |
| OpenAI: luna draft + gpt-6-astra verify | $8.98-9.13 | $4.49-4.56 | $1.80-1.83 | $0.898-0.913 | $7.78-7.91 |
| OpenAI: gpt-6.1-terra only (mid, reference) | $2.25-3.75 | $1.13-1.88 | $0.450-0.750 | $0.225-0.375 | $1.95-3.25 |
| Google: gemini-3.5-flash-lite only | $0.975-1.58 | $0.488-0.788 | $0.195-0.315 | $0.098-0.158 | $0.845-1.37 |
| Google: gemini-3.1-flash-lite only | $0.638-1.05 | $0.319-0.525 | $0.128-0.210 | $0.064-0.105 | $0.553-0.910 |
| Google: 3.5-flash-lite draft + gemini-3.8-flash verify (price to 2026-12-31) | $1.63-2.23 | $0.816-1.12 | $0.326-0.446 | $0.163-0.223 | $1.41-1.93 |
| Google: same, 3.8-flash at 2027-01-01 price | $2.29-2.89 | $1.14-1.44 | $0.458-0.578 | $0.229-0.289 | $1.98-2.50 |
| Google: 3.5-flash-lite draft + gemini-4-argon verify (price to 2026-11-30) | $2.73-3.33 | $1.36-1.66 | $0.545-0.665 | $0.273-0.333 | $2.36-2.88 |
| Google: same, argon at 2026-12-01 price | $4.48-5.08 | $2.24-2.54 | $0.895-1.02 | $0.448-0.508 | $3.88-4.40 |
| Google: gemini-3.8-flash only (mid, to 2026-12-31) | $1.69-2.81 | $0.844-1.41 | $0.338-0.563 | $0.169-0.281 | $1.46-2.44 |
| Google: gemini-3.8-flash only (mid, from 2027-01-01) | $3.38-5.63 | $1.69-2.81 | $0.675-1.13 | $0.338-0.563 | $2.93-4.88 |

- Verification pass alone, 500 questions, standard: Sonnet 5.5 or gpt-6.1-sol or Argon (to 2026-11-30) $1.75; Opus 5.5 or Argon (from 2026-12-01) $3.50; gpt-6-astra $8.75; gemini-3.8-flash $0.66 (to 2026-12-31) or $1.31 after.
- Most expensive row (luna draft + astra verify), high case: launch $9.13 plus one month of weekly runs $7.91 is about $17.03, below the USD 22.27 budget. Every other row is lower.
- Input sensitivity: each extra 1,000 input tokens per request over 500 requests adds $0.05 (Haiku 5.5, luna), $0.15 (3.5 Flash-Lite), $0.38 (3.8 Flash), $1.00 (Sonnet 5.5, gpt-6.1-sol, Argon to 2026-11-30).
- Output sensitivity (hidden reasoning): each extra 1,000 output tokens per request over 500 requests adds $0.25 (Haiku 5.5, luna), $1.25 (3.5 Flash-Lite), $1.88 (3.8 Flash), $5.00 (Sonnet 5.5, gpt-6.1-sol, Argon to 2026-11-30), $10.00 (Opus 5.5), $25.00 (astra).

## 4. Implications for skillassert (facts and constraints)

1. At the assumed token counts every pipeline above costs under USD 10 for the 500-question launch and under USD 8 per month at 100 questions per week. Hidden reasoning tokens and longer prompts change this (items 2 and 3).
2. Reasoning tokens are billed as output and are on by default: Haiku 5.5 effort default medium; Sonnet 5.5 effort default high and `thinking: disabled` returns 400 (off via `between_tools` or low effort); gpt-6-luna and gpt-6.1-terra default medium with `none` available, gpt-6.1-sol lowest is `low`; Gemini 3.8 Flash default medium with lowest `low`, 3.5 Flash-Lite default minimal [A1 O9 G9]. The 150-token verification output only holds if reasoning is turned down.
3. Token counts differ by provider: Anthropic states its newer tokenizer produces about 30% more tokens than earlier Claude models for the same text [A2]. The 600 to 900 output estimate was not measured on any provider.
4. Gemini list prices double on 2026-12-01 (Argon) and 2027-01-01 (3.8 and 3.6 Flash). The ongoing 100-per-week workload will run at the higher prices. The pricing page lists no scheduled change for the Flash-Lite models.
5. Gemini free tier versus the app: the terms restrict API clients offered to users in the EEA, Switzerland or the UK to Paid Services, while a developer inside the EEA already gets paid-service data terms on free quota. Whether an admin-triggered server batch counts as "making API Clients available" is not defined in the terms.
6. EU processing guarantees exist only on Google Cloud's Gemini platform (separate product, about +10%) and OpenAI's EU endpoint (sales-gated, +10%, needs a retention amendment). Anthropic's first-party API has no EU option, and its Batches API is not offered on Bedrock or Vertex.
7. Schema enforcement differs: minItems/maxItems (for 4 to 5 options) can be enforced in the OpenAI and Gemini schemas but not in Anthropic's (only 0 or 1), so the option count needs app-side validation on Anthropic. A rule such as "exactly one correct answer" needs app-side validation on all three.
8. Caching on a 1,500-token prompt is available on Anthropic and OpenAI and unavailable on four of the Gemini models (4,096 minimum). The dollar saving is under $0.07 per 500 requests for the small models and about $1.42 for Sonnet 5.5 or gpt-6.1-sol.
9. Batch gives 50% everywhere, but turnaround is up to 24 h on all three (Anthropic: most batches within 1 h). Standard calls fit inside the entry-tier limits above; a same-evening batch depends on the batch finishing early.
10. Anthropic Haiku 5.5 can return `stop_reason: refusal` from safety classifiers and has no server-side fallback [A1]; structured output may then not match the schema.
11. A ChatGPT or Codex subscription does not include API usage. Plus has no API credits; Pro and Business can claim monthly API credits (up to $500) that offset normal per-token API billing and are separate from signed-in Codex limits. The developer's plan tier is not known here.
12. Entry limits that were not published: Anthropic Evaluation tier, Gemini per-model RPM/TPM/RPD, OpenAI Free-tier model limits. A new account may sit below the tables above.

## 5. Could not verify

- Anthropic: Evaluation-tier limits; size, expiry and card rule of the free starter credit; minimum credit purchase; training-data policy; which EU regions on Bedrock or Vertex serve which Claude model.
- OpenAI: any free credit for new API accounts; what the Free tier can call and at what limits; whether EU residency is self-serve (docs say contact sales); the help-center articles on ChatGPT versus API billing and on ChatGPT Plus (HTTP 403 to curl and to the fetch tool, so the Plus/API answer rests on O10 to O12); whether Chat Completions is covered by subscriber credits; whether the developer's own organization shows the complimentary-token offer; the data-sharing page was read from a 2026-10-17 archive copy, not live.
- Google: per-model RPM, TPM and RPD for Free and Tier 1 (docs point to AI Studio; a third-party blog's figures were not used); implicit-cache minimum for the Flash-Lite models; explicit-cache minimums; whether Batch accepts structured-output config (docs say each request can carry other configurations, no example shown); processing location on the Developer API; whether `generateContent` batch works on EU Cloud endpoints.
- Quality: no source compares question quality or answer-verification accuracy across these models. No benchmark was gathered.
- Latency per request, real token counts per question, and the meaning of Gemini's "billing tier cap" ($250 at Tier 1).
- Anthropic skill vs live page: Sonnet 5.5 cache read $0.20 vs $0.10 (live page used; skill table dated 2026-10-06).

## 6. Sources (all read 2026-10-29 unless noted)

- A1 `claude-api` skill: Current Models table (cached 2026-10-06), shared/prompt-caching.md, shared/tool-use-concepts.md, shared/platform-availability.md, typescript/claude-api/batches.md.
- A2 https://platform.claude.com/docs/en/about-claude/pricing.md ; A3 https://platform.claude.com/docs/en/api/rate-limits.md ; A4 https://platform.claude.com/docs/en/build-with-claude/structured-outputs.md ; A5 https://platform.claude.com/docs/en/build-with-claude/batch-processing.md ; A6 https://platform.claude.com/docs/en/build-with-claude/prompt-caching.md ; A7 https://platform.claude.com/docs/en/manage-claude/data-residency.md
- O1 https://developers.openai.com/api/docs/pricing.md ; O2 https://developers.openai.com/api/docs/models/gpt-6-luna.md (also gpt-6.1-terra, gpt-6.1-sol) ; O3 https://developers.openai.com/api/docs/guides/latest-model/gpt-6-astra.md ; O4 .../guides/structured-outputs.md ; O5 .../guides/batch.md ; O6 .../guides/prompt-caching.md ; O7 .../guides/rate-limits.md ; O8 .../guides/your-data.md ; O9 .../guides/reasoning.md ; O10 .../guides/chatgpt-api-credits.md ; O11 https://learn.chatgpt.com/docs/pricing.md ; O12 https://learn.chatgpt.com/docs/auth.md ; O13 https://help.openai.com/en/articles/10306912-sharing-feedback-evaluation-and-fine-tuning-data-and-api-inputs-and-outputs-with-openai (read via web.archive.org snapshot 20261017151648)
- G1 https://ai.google.dev/gemini-api/docs/pricing ; G2 .../models ; G3 .../structured-output and .../migrate-to-interactions ; G4 .../batch-api ; G5 .../caching (updated 2026-10-23) ; G6 .../rate-limits ; G7 .../billing (updated 2026-09-28) ; G8 https://ai.google.dev/gemini-api/terms (effective 2026-03-23) ; G9 .../thinking ; G10 https://docs.cloud.google.com/gemini-enterprise-agent-platform/resources/data-residency and https://cloud.google.com/model-garden/pricing
- X1 https://www.ecb.europa.eu/stats/eurofxref/eurofxref-daily.xml (rate dated 2026-10-28)
