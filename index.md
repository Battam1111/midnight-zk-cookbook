---
layout: default
title: Midnight ZK Cookbook
---

# The ZK Cookbook

**17 production-ready tutorials** spanning Midnight Compact, Aleo Leo, Noir (Aztec), RISC Zero zkVM, and cross-ecosystem architecture. By **Yanjun Chen** ([Battam1111](https://github.com/Battam1111)). Total: ~54,200 words.

[![GitHub Sponsors](https://img.shields.io/github/sponsors/Battam1111?style=for-the-badge)](https://github.com/sponsors/Battam1111) [![Pricing](https://img.shields.io/badge/Pricing-purchase-purple?style=for-the-badge)](pricing.html) [![Bounty Radar](https://img.shields.io/badge/Live-bounty%20radar-blue?style=for-the-badge)](https://battam1111.github.io/bounty-radar-data/)

---

## Why this exists

The official ZK toolchain docs cover the language and the build flow. This cookbook fills the **gap between hello-world and shipping a real ZK contract** — across five major ecosystems, with cross-references so you can compare patterns.

Every example uses real, verified syntax from each ecosystem's official reference. Where compiler versions matter, it's flagged in the tutorial.

---

## Paid offerings

- **[$15 ZK Cookbook Bundle](pricing.html#bundle)** — every tutorial as a single Markdown file + companion code repos
- **[$29 ZK Bounty Hunter Playbook](pricing.html#playbook)** — 7-chapter PDF on how to find and win ZK bounties (50+ pages)
- **[$49 Cookbook + zk-doctor Pro License](pricing.html#pro)** — bundle + zk-pipeline-doctor Pro features
- **[$99 ZK Project Pre-Audit](pricing.html#audit)** — expert-narrated 24-hour audit of your repo · [sample](https://battam1111.github.io/bounty-radar-data/audits/sample.html)
- **[$19/97/497 Bounty Radar](pricing.html#radar)** — live ZK bounty feed (3 tiers)

All payment via [Polar.sh](https://polar.sh) (Merchant of Record). 14-day money-back guarantee.

---

## Tutorials by ecosystem

### Midnight (Compact)

- **[289](tutorials/289.html)** — ~3,497 words
  <br>_- a Map when the contract itself owns and mutates on-chain key/value state; and - a Merkle tree when you want to prove membership against a committed root, usually to keep per-user data off-chain and proofs compact._
- **[291](tutorials/291.html)** — ~3,513 words
  <br>_In Midnight, a witness is an off-chain callback that supplies private or externally computed data to a circuit. The circuit does not trust the witness implementation itself; it only proves statements about the witness output under constrain…_
- **[295](tutorials/295.html)** — ~3,170 words
  <br>_If you use ownPublicKey() as an authorization check in a Midnight Compact contract, you are not proving “the caller controls this public key.” You are only proving “the prover supplied this public key as a private circuit input.”_
- **[296](tutorials/296.html)** — ~3,755 words
  <br>_This tutorial designs a DAO-style commit/reveal voting system for Midnight in Compact, with six required features:_
- **[303-multi-party-private-state-and-contracts-between-two-users](tutorials/303-multi-party-private-state-and-contracts-between-two-users.html)** — ~3,880 words
  <br>_The official two-party examples on Midnight are a good starting point, but the core pattern scales to N participants if you make three design changes:_
- **[304](tutorials/304.html)** — ~3,303 words
  <br>_Midnight does not ship a built-in oracle mechanism. That is not an omission so much as a design constraint: Compact circuits prove computation over explicit inputs and ledger state, while off-chain data remains off-chain unless your applica…_
- **[312](tutorials/312.html)** — ~3,636 words
  <br>_If you want reliable tests for Midnight Compact contracts, split your suite into two layers:_

### Aleo (Leo)

- **[Your First Aleo Contract in Leo: Mappings, Records, and Transitions](tutorials/aleo-first-contract.html)** — ~3,591 words
  <br>_This tutorial is written against the Aleo reference primer fetched on 2026-05-20 and the mainnet snarkVM README. The Leo documentation URLs in the primer resolve to “Page Not Found,” so this guide is intentionally strict about what is and i…_
- **[Testing Aleo Programs Locally with snarkOS and snarkVM](tutorials/aleo-program-testing.html)** — ~3,054 words
  <br>_If you are evaluating whether to ship an Aleo program, local testing should happen at two distinct layers:_

### Noir (Aztec)

- **[Writing Your First Noir Circuit: Fields, Constraints, and Proofs](tutorials/noir-circuit-basics.html)** — ~3,330 words
  <br>_This tutorial is written against the Noir documentation pages fetched on 2026-05-20, specifically the dev docs that note the latest stable version as v1.0.0-beta.21. Where behavior may differ across releases, you should verify against your …_
- **[Building a Zero-Knowledge Identity Proof in Noir](tutorials/noir-identity-proof.html)** — ~2,328 words
  <br>_This tutorial builds a small but real Noir circuit for private identity proofs:_
- **[Testing and Debugging Noir Circuits: The Honest Workflow](tutorials/noir-testing-debugging.html)** — ~3,369 words
  <br>_This tutorial is written against the Noir documentation snapshot fetched 2026-05-20, specifically the dev docs that also point readers to v1.0.0-beta.21 as the latest version in the navigation. Everything below stays within syntax and conce…_

### RISC Zero (zkVM)

- **[RISC Zero zkVM: Running Existing Rust Code Inside a STARK](tutorials/risc0-zkvm-rust.html)** — ~3,053 words
  <br>_If you already write Rust and you’re curious about zero-knowledge systems, Risc0 is one of the most ergonomic entry points because it lets you prove execution of ordinary Rust code instead of rewriting everything as arithmetic constraints._

### Cross-ecosystem

- **[Aztec Accounts vs Aleo Records — Privacy Models Deep Dive](tutorials/aztec-aleo-privacy-models.html)** — ~3,065 words
  <br>_Below is a protocol-engineering comparison of Aztec and Aleo as they actually model private state today: both are hybrids, but they put the “privacy primitive” in very different places._
- **[ZK Language Comparison: Compact vs Leo vs Noir vs Cairo (2026)](tutorials/zk-language-comparison.html)** — ~3,169 words
  <br>_- Compact, Leo, and Cairo are primarily chain/application languages. - Noir is primarily a circuit language that becomes a chain language only when paired with Aztec’s contract framework._
- **[ZK Rollup Architecture Explained: zkSync vs Linea vs Polygon zkEVM (2026)](tutorials/zk-rollup-architecture.html)** — ~3,078 words
  <br>_Below is an engineering-oriented overview. One constraint up front: I do not have live web access, so I cannot honestly claim verified 2026 dashboard values. Where you asked for “real production numbers,” I use publicly reported production …_

### Dev tools / CI

- **[Add zk-doctor to your ZK project's CI in 5 minutes](tutorials/zk-doctor-github-action.html)** — ~1,409 words
  <br>_If you write zero-knowledge code in Compact, Leo, Noir, Cairo, or Risc0 Rust, you live with a familiar problem: you can't easily tell, at a glance, whether a ZK repo is production-ready. The README looks fine. Tests pass. CI is green. And t…_

---

## Companion tools (all open-source, MIT)

- **[zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor)** — CLI: 6-detector ZK project health audit
- **[zk-doctor-action](https://github.com/Battam1111/zk-doctor-action)** — 5-minute GitHub Action wrapping the CLI
- **[bounty-radar-data](https://github.com/Battam1111/bounty-radar-data)** — live ZK bounty feed (JSON + RSS + HTML)
- **[bounty-radar-mcp](https://github.com/Battam1111/bounty-radar-mcp)** — MCP server, query the radar from Claude / Cursor / any MCP client
- **[zk-doctor-bot](https://github.com/Battam1111/zk-doctor-bot)** — GitHub App for PR-time reviews (in beta)

---

## Support

If these tutorials saved you time, support the work:

- Star [the source repo](https://github.com/Battam1111/midnight-zk-cookbook) on GitHub
- [Sponsor on GitHub](https://github.com/sponsors/Battam1111) (one-time or recurring)
- [Buy a tier on Polar](pricing.html)
- Share with your ZK community

---

*Cross-reference against each ecosystem's official docs before using in production. Compiler-version-specific notes flagged inline in each tutorial.*

## 中文教程 (Chinese tutorials)

- [303-multi-party-private-state-and-contracts-between-two-users](tutorials-cn/303-multi-party-private-state-and-contracts-between-two-users.html) — 1043 字
- [312](tutorials-cn/312.html) — 1298 字
- [5 分钟把 zk-doctor 接入你的 ZK 项目 CI](tutorials-cn/zk-doctor-github-action.html) — 926 字
- [ZK 语言对比：Compact vs Leo vs Noir vs Cairo（2026）](tutorials-cn/zk-language-comparison.html) — 1954 字
