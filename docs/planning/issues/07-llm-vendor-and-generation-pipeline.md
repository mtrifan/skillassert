# 07: LLM vendor and generation pipeline details

Type: grilling
Status: resolved
Claimed by: this session, 2026-10-29 (round 4 grilling)
Blocked by: 02

## Question

Using research 02: which vendor and models draft and verify questions; the structured-output schema; batch size
and concurrency under new-account rate limits; de-duplication against existing questions; a per-batch cost cap;
prompt versioning so regenerated questions can be traced to the prompt that made them. Resolve with grilling and
domain-modeling; record the answer here and patch design.md.

## Answer

Resolved 2026-10-29 in round 4 of the grilling log (Q36 to Q40). The owner is on ChatGPT Plus, which carries no
API credits, so cost does not separate Anthropic and OpenAI. Decision: the generator port gets two adapters,
Anthropic (claude-haiku-5-5 draft, claude-sonnet-5-5 verify) and OpenAI (gpt-6-luna draft, gpt-6.1-sol verify);
the first batch for one topic and tier runs 20 questions through each, reviewed blind, and the winner becomes the
default with the other as fallback. Five questions per call with strict JSON-schema output, reasoning effort at
the lowest setting, hard output limits, cache-friendly prompt layout, synchronous calls three in flight, no batch
API. Cost caps: $2 per batch, $10 per month, enforced in the service from recorded usage, plus a vendor-console
spend limit. Written into design.md section 11.
