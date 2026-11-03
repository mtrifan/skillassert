# 02: LLM generation cost and capabilities (facts as of 2026-10-29)

Type: research
Status: resolved
Blocked by: none
Claimed by: research subagent, 2026-10-29

## Question

For generating multiple-choice questions with explanations (about 500 before launch, then about 100 a week), what
would the current small and mid-size models from Anthropic, OpenAI and Google cost per question; which support
JSON-schema structured output; what batch discounts, rate limits on new accounts, free credits and EU data-residency
options exist; and what does a two-pass pipeline (cheap model drafts, stronger model verifies correctness) cost?

## Why it is on the map

Round 2 Q19 and Q20 decide the generation pipeline and provider. Prices and model names change too fast to quote
from memory.

## Answer

Resolved 2026-10-29. Full report with sources and arithmetic: [research/02-llm-generation-cost-and-capabilities.md](../research/02-llm-generation-cost-and-capabilities.md).

- Cost is not the deciding factor. At the brief's token counts the 500-question launch batch costs $0.23 to $0.38
  on a small model (claude-haiku-5-5 or gpt-6-luna, both $0.10/$0.50 per 1M in/out) and about $2 with a
  Sonnet 5.5 or gpt-6.1-sol verify pass; 100 questions a week with verify is about $0.40. Batch APIs halve it.
  Worst case found is about $17 for launch plus a month, under the EUR 20 (USD 22.27) budget.
- All three vendors offer native JSON-schema output and a 50% batch discount. Prompt caching minimums: Anthropic
  512 tokens, OpenAI 1,024, Gemini 4,096 on the relevant models, so a 1,500-token system prompt caches on the
  first two only.
- Entry-tier rate limits fit the launch batch in a minute or two of quota on Anthropic and OpenAI; Gemini
  per-model limits are not published.
- Gemini's free tier is unusable for a live EEA-facing app (terms: "only Paid Services"); paid needs billing and
  a $5 prepay. Gemini Flash-Lite also costs 3 to 5x the other small models per output token.
- EU residency: none first-party at Anthropic; OpenAI eu endpoint +10% and sales-gated; Gemini only via Google
  Cloud. Irrelevant for this pipeline: it sends topic briefs and existing question prompts, never learner data.
- Biggest cost risk: reasoning tokens are billed as output and on by default; each extra 1,000 output tokens per
  request over 500 requests adds $0.25 on small models and $5 on the mid models. Effort must be set low or off for
  drafting and verifying.
- Subscription benefit: API usage is billed separately from every ChatGPT plan. Pro subscribers can claim $100,
  $200 or $500 of API credits per month by tier, Business $20 per Standard seat or $100 per Premium seat capped at
  $500 per workspace; Plus, Go, Free and Enterprise get none. Conditions: plan at least 7 days old, same email owns
  the subscription and the API organization, credits expire each billing cycle, Responses API named (OpenAI docs,
  chatgpt-api-credits guide, read 2026-10-29). A data-sharing opt-in can add 250,000 complimentary tokens a day on
  the gpt-6-luna group at the Build tier, worth at most $0.125 a day (archived page, not live). Whether the user's
  plan qualifies is asked in round 4.
- Unverified: Anthropic free starter credit size and card rule; OpenAI new-account credits; Gemini RPM/TPM; any
  quality comparison between the models for this task.
