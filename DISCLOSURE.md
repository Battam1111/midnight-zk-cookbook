# AI Use Disclosure

This repository previously contained ~17 technical tutorials about zero-knowledge programming
across multiple ecosystems (Midnight Compact, Aleo Leo, Noir, RISC Zero, etc.). **A substantial
portion of that content was generated with the assistance of a large language model (LLM) and
had not been line-by-line verified by a human expert.** Several tutorials shipped with
visible LLM conversational tails (e.g. "If you want, I can turn this into..."), and at least
one tutorial explicitly disclosed that the model "did not have live web access" and so could
not verify numbers cited in the text.

That is a quality and ethics problem. Concretely:

- LLM-generated technical content for narrow specialist domains is known to hallucinate at
  **31% in ordinary use and up to 60% in complex domains** (peer-reviewed surveys). ZK is one
  of the most complex domains in software.
- Publishing such content under an author's real name, without explicit disclosure, is
  inconsistent with the academic-integrity standards expected of graduate researchers.

**As of 2026-05-23 the cookbook has been rolled back.** The previous tutorials have been
removed pending human verification. Going forward this repository will:

1. Disclose **every** use of AI assistance, inline at the top of each affected page.
2. Not republish any tutorial unless the author can vouch personally for the correctness of
   every code sample and technical claim in it.
3. Treat AI as a drafting / proofreading aid, not as a primary author.

If you arrived here from a previous link that 404'd, sorry — that's intentional. We would
rather have nothing on a page than content we cannot stand behind.

The open-source companion tools in this family — `zk-pipeline-doctor`, `zk-doctor-action`,
`bounty-radar-data`, `bounty-radar-mcp` — **are real code, with real tests, and continue to
be maintained**. They are unaffected by this rollback.

— Battam1111, 2026-05-23
