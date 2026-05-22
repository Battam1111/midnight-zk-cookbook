# AI Use Disclosure

## How tutorials in this cookbook are authored (post-2026-05-23)

Tutorials are drafted in **Claude Code sessions** by the author
([Battam1111](https://github.com/Battam1111)) using:

- Claude (Anthropic's assistant) as a drafting partner
- Sub-agents for parallel research (web search, GitHub repo verification)
- Live web search for up-to-date references against each ZK ecosystem's
  official documentation

Each tutorial is then **personally reviewed line-by-line** by the author,
who compiles every code sample in the relevant toolchain before publishing.

## Automated linter

Before any tutorial reaches the live site, it must pass an automated
**AI-tell linter** that scans for visible LLM conversational patterns:

- "If you want, I can..." (chatbot offer-to-continue)
- "Let me know if..." (chatbot follow-up offer)
- "Would you like me to..." (chatbot follow-up offer)
- "I do not have live web access..." (model self-disclosure)
- "I cannot honestly claim..." (model self-disclosure)
- "As an AI..." (model identity disclosure)

The linter is open source — see
[`bounty-agent/src/bounty/content_pipeline/ai_tell_linter.py`](https://github.com/Battam1111)
in the private engine repo. The same patterns are blocked at publish time;
they cannot reach this cookbook even if a draft slips past review.

## What happened on 2026-05-22

Earlier on the 22nd, an automated Frontier API pipeline drafted and
published 17 tutorials in three days. Three of those tutorials contained
visible LLM conversational tails ("If you want, I can turn this into a
research-note version..."). This was a quality-control failure: the
pipeline ran without an automated lint and the human reviewer was
bulk-processing rather than line-reviewing.

The infrastructure has been corrected as of 2026-05-23:

1. The Frontier-API auto-publish pipeline is **disabled by default**
   (config flag `BOUNTY_AI_GENERATION_ENABLED=false`).
2. Tutorials are now drafted in Claude Code sessions where a human is in
   the loop in real time, and sub-agents can verify code against live
   repos and official docs.
3. The AI-tell linter runs automatically before any tutorial publishes.
4. The publish pipeline requires two environment variables and a manual
   flag, deliberately preventing background jobs from publishing without
   human intent.

The three problem tutorials had their conversational tails stripped on
2026-05-23; the underlying technical content remains. If you spot any
remaining accuracy issue in any tutorial, please open an issue against
this repo — corrections are taken seriously.

## What Frontier API still does

Internal-only scoring within `bounty-agent`:

- Bounty go/no-go decisions (numeric scores stored in SQLite)
- Spam classification

These outputs never become public prose, so hallucination risk is bounded.

— Battam1111
