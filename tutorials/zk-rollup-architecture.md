---
layout: default
title: "ZK Rollup Architecture Explained: zkSync vs Linea vs Polygon zkEVM (2026)"
slug: zk-rollup-architecture
ecosystem: cross
---

# ZK Rollup Architecture Explained: zkSync vs Linea vs Polygon zkEVM (2026)

Below is an engineering-oriented overview. One constraint up front: I do **not** have live web access, so I cannot honestly claim verified **2026** dashboard values. Where you asked for “real production numbers,” I use **publicly reported production ranges available through mid-2024** from sources such as **L2BEAT, growthepie, and project explorers**, and I label them as such rather than inventing 2026 figures.

# ZK rollups vs optimistic rollups: an architectural overview

Rollups all do the same high-level trick: execute transactions off Ethereum, but keep Ethereum as the settlement layer. The two dominant families differ in **how Ethereum is convinced the offchain execution was correct**.

- **Optimistic rollups** assume batches are correct unless challenged.
- **ZK rollups** prove batches are correct up front with validity proofs.

That difference propagates into everything else: latency, withdrawal UX, prover economics, compatibility, and decentralization strategy.

## 1. Rollup taxonomy

### 1.1 Optimistic vs ZK

At a high level:

```text
Optimistic rollup:
  L2 posts tx data + new state commitment to L1
  L1 accepts "optimistically"
  Anyone may challenge during dispute window
  Finality delayed by challenge period

ZK rollup:
  L2 posts tx data + new state commitment + validity proof
  L1 verifies proof
  Finality follows proof acceptance on L1
  No challenge window for correctness
```

The key engineering distinction is the **cost model**:

- **Optimistic**: cheap to generate batches, expensive only in disputed cases.
- **ZK**: expensive to generate proofs every batch, cheap for L1 to verify.

So optimistic systems externalize correctness cost into a dispute game; ZK systems internalize it into continuous proving.

### 1.2 Sequencer architectures

Most production rollups today, including **zkSync Era, Linea, and Polygon zkEVM**, have historically used a **single sequencer/operator path** for ordering transactions. That means:

- low latency
- simple UX
- centralized censorship/fair-ordering risk
- decentralization deferred to later roadmap phases

Typical components:

```text
User -> Sequencer -> Executor -> Batcher -> DA publisher -> Prover -> L1 Verifier
```

Roles:

- **Sequencer**: orders transactions, gives users soft confirmations
- **Executor**: runs the EVM/VM and computes new state
- **Batcher**: compresses transactions/state diffs and posts data to Ethereum
- **Prover**: generates the validity proof
- **Verifier contract**: checks proof and accepts the new state root

The important nuance is that **sequencing** and **proving** are orthogonal. A rollup can have:
- centralized sequencing + decentralized proving
- decentralized sequencing + centralized proving
- both centralized
- both decentralized

In practice, production ZK rollups have usually decentralized the **verification** first, not the **sequencing**.

### 1.3 Data availability

A system is a **rollup** in the strict sense only if the data needed to reconstruct state is available on Ethereum.

That means the L2 must publish either:
- full transaction calldata, or
- equivalent state-diff/input data sufficient for reconstruction,
- increasingly via **EIP-4844 blobs** rather than calldata

If data is kept offchain by a committee, the system is closer to a **validium** or **volition** mode.

For the three systems here:

- **zkSync Era**: Ethereum DA, with strong emphasis on compressed state diffs and bytecode/data efficiency
- **Linea**: Ethereum DA
- **Polygon zkEVM**: Ethereum DA

Post-4844, the architectural story changed materially: batch cost is now dominated less by calldata pricing and more by blob inclusion strategy, proof cadence, and compression efficiency.

---

## 2. Why ZK rollups differ operationally from optimistic rollups

Optimistic rollups and ZK rollups both inherit Ethereum security for settlement, but they expose different latency surfaces.

### Optimistic rollups

Operational path:

```text
1. Sequencer executes txs and gives user an L2 receipt
2. Batcher posts tx data and state commitment to L1
3. State is "pending" economically
4. Challenge window passes (commonly ~7 days)
5. Withdrawal finalizes
```

Implications:

- low proving overhead
- good EVM compatibility
- delayed canonical finality
- native withdrawals back to L1 are slow unless liquidity providers front capital

### ZK rollups

Operational path:

```text
1. Sequencer executes txs and gives user an L2 receipt
2. Batcher posts DA payload to L1
3. Prover generates validity proof
4. Verifier contract accepts proof
5. State becomes finalized on L1
```

Implications:

- no fraud window for correctness
- faster native withdrawals
- higher fixed proving cost
- compatibility depends on how faithfully the circuit models the EVM

This is why application teams often pick ZK for:
- exchange flows needing faster exit guarantees
- cross-chain bridging where week-long withdrawal delays are unacceptable
- systems sensitive to reorg/dispute uncertainty

But optimistic rollups still have advantages where:
- maximal EVM compatibility matters
- prover economics are too costly
- throughput targets can be met without validity proofs

---

## 3. The three production ZK rollups

## 3.1 zkSync Era

**zkSync Era** is architecturally distinctive because it is not just “EVM inside a proof.” It uses **EraVM** and a custom compilation/bootloader model, which improves ZK-friendliness but introduces semantic differences versus Ethereum.

Core traits:
- validity rollup on Ethereum
- custom VM design oriented toward efficient proving
- compressed state-diff publishing
- proof system branded **Boojum**
- stronger tradeoff toward ZK efficiency than strict bytecode equivalence

### Boojum

Boojum is Matter Labs’ proving stack for Era. The important architectural point is not the brand name but the design goal:

- make proof generation practical at production scale
- push proving onto commodity hardware where possible
- optimize recursion and circuit engineering around the actual Era execution model

Because Era is not strict L1 bytecode equivalence, Boojum benefits from a VM that is more tractable to prove than raw Ethereum semantics.

## 3.2 Linea

**Linea** takes the opposite posture: it aims to feel very close to Ethereum for developers. ConsenSys positioned it around high compatibility with existing Solidity tooling and MetaMask-native distribution.

Core traits:
- zkEVM rollup on Ethereum
- compatibility-first design
- Ethereum DA
- prover stack often described in engineering materials as involving **Vortex** plus circuit tooling such as **gnark**
- tends to prioritize dev portability over aggressive VM redesign

### Vortex

Terminology around Linea’s proving internals can vary by document: some materials describe the proof stack via circuit frameworks, others via arithmetization/recursion layers. The important engineering takeaway is that Linea’s prover is designed around **high EVM fidelity**, which generally means:
- more complex circuits than a custom ZK VM
- potentially heavier proving requirements
- fewer semantic surprises for existing contracts

## 3.3 Polygon zkEVM

**Polygon zkEVM** sits between Ethereum equivalence ambition and Polygon’s broader proving research stack.

Core traits:
- zkEVM rollup settled on Ethereum
- strong compatibility target
- Ethereum DA
- production architecture historically associated with Polygon Hermez + Polygon Zero proving work
- Polygon’s proving roadmap is strongly associated with the **Plonky** family

### Plonky3: important caveat

You asked specifically for **Plonky3**. The right engineering caveat is:

- **Polygon’s ecosystem is deeply associated with the Plonky proving family**
- **Polygon zkEVM mainnet production proofs were historically built around earlier-generation components rather than a clean “Plonky3 everywhere” production statement, at least in the public material available through mid-2024**
- so it is accurate to discuss **Plonky3 as Polygon’s proving direction/framework lineage**, but not to overclaim a completed production cutover without live verification

That matters because researchers often blur:
- proving framework
- circuit library
- recursion layer
- actual production prover used by the chain at a given date

---

## 4. EVM compatibility: Type 1-4 and tradeoffs

Vitalik’s type taxonomy is still a useful shorthand, though real systems often land “between” categories.

```text
Type 1: Ethereum-equivalent
  Proves Ethereum blocks almost exactly as-is

Type 2: Fully EVM-equivalent
  Very high compatibility, minor implementation differences

Type 3: Almost EVM-equivalent
  Small semantic/tooling differences

Type 4: High-level-language compatible
  Solidity/Vyper friendly, but not bytecode equivalent
```

### Rough placement of the three systems

#### Linea: closest to Type 2
Linea is best understood as a **Type 2-ish** design:
- high tooling portability
- close execution semantics
- easier migration for existing dapps
- heavier proving burden

#### Polygon zkEVM: Type 2 to Type 3
Polygon zkEVM has generally been positioned near **Type 2**, but in practice many zkEVMs have enough edge-case differences to look **Type 2.5 / Type 3** operationally:
- strong Solidity compatibility
- some differences around gas accounting, precompiles, or obscure EVM behavior
- better developer portability than custom VMs

#### zkSync Era: Type 3 to Type 4
zkSync Era is the least “literal Ethereum” of the three:
- Solidity support is good
- but execution runs through EraVM and custom compilation paths
- bytecode assumptions and low-level tricks may not port cleanly
- stronger ZK efficiency, weaker raw equivalence

### The tradeoff

Higher equivalence gives:
- easier contract migration
- more reliable bytecode-level tooling
- fewer edge-case surprises

Lower equivalence gives:
- simpler or cheaper proving
- better long-run scalability knobs
- room for VM features designed around proof efficiency

For app teams, this is not abstract. If you rely on:
- advanced bytecode patterns
- L1-exact gas behavior
- esoteric precompile assumptions
- custom debuggers / low-level tracing

then Type 2-ish systems are much less risky.

If you mainly deploy conventional Solidity applications and care more about fee efficiency and future throughput, a more custom VM may be acceptable.

---

## 5. End-to-end flow: sequencer, batcher, prover, verifier

## 5.1 ZK rollup flow

```text
                 +---------------------+
Users/Builders ->|      Sequencer      |-- soft confirmation -->
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 |  Executor / VM      |
                 |  tx execution       |
                 |  state transitions  |
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 |       Batcher       |
                 | compress txs/diffs  |
                 | build L1 payload    |
                 +-----+----------+----+
                       |          |
             DA to L1  |          | witness / traces
                       v          v
            +----------------+  +-------------------+
            | Ethereum DA    |  |      Prover       |
            | calldata/blobs |  | circuit generation|
            +--------+-------+  | recursion/agg     |
                     |          +---------+---------+
                     |                    |
                     +--------------------+
                                          |
                                          v
                              +-----------------------+
                              | L1 Verifier Contract  |
                              | verify validity proof |
                              | accept new state root |
                              +-----------+-----------+
                                          |
                                          v
                                  Finalized L2 state
```

### Latency buckets in a ZK rollup

The real latency is the sum of:
1. **sequencer inclusion time**
2. **batch formation time**
3. **L1 data publication time**
4. **proof generation time**
5. **proof posting / L1 confirmation time**

Users often conflate “my tx was included quickly” with “it is finalized on Ethereum.” They are different.

## 5.2 Optimistic rollup flow

```text
Users -> Sequencer -> Execution -> Batcher -> L1 assertion/data post
                                            |
                                            v
                                  Challenge / dispute window
                                            |
                                  no valid fraud proof?
                                            |
                                            v
                                       Finalization
```

The crucial difference is that optimistic systems replace continuous proving with a **dispute game**. This is why they can be operationally simpler but have much worse native withdrawal latency.

---

## 6. Proving systems: Boojum, Vortex, Plonky3

## Boojum / zkSync Era

Engineering emphasis:
- custom VM-friendly circuit design
- recursion to aggregate many execution traces
- lower prover cost through architecture co-design

Best for:
- teams willing to tolerate nontrivial execution differences
- long-run scaling via VM redesign

## Vortex / Linea

Engineering emphasis:
- proving high-fidelity EVM execution
- preserving developer expectations from Ethereum
- tighter integration with existing Ethereum tooling stack

Best for:
- application portability
- enterprise or mainstream teams valuing low migration friction

## Plonky3 / Polygon ecosystem direction

Engineering emphasis:
- very fast proving-oriented polynomial IOP machinery and recursion framework lineage
- modularity for future proof systems and aggregation layers
- strong research velocity from Polygon Zero

Best interpreted as:
- an indicator of Polygon’s proving direction
- not a reason to ignore the exact production prover actually running the chain on a given date

For researchers, the main observation is this: the three systems are not merely swapping proof libraries; they represent different positions on the spectrum of **equivalence vs proof efficiency**.

---

## 7. Performance numbers: what public dashboards showed through mid-2024

Again, these are **not 2026-verified values**. They are the best non-marketing production ranges visible on public dashboards through my knowledge cutoff.

### Usage and throughput

Using **growthepie**, **L2BEAT**, and chain explorers through mid-2024:

- **zkSync Era**
  - typically the busiest of the three
  - often in the range of **~0.5M to 1.0M daily transactions** on active periods
  - peak days could move above that depending on campaigns and spam
  - TVS/TVL fluctuated widely but was often **hundreds of millions to above $1B**

- **Linea**
  - meaningful production usage but below Era
  - commonly **~0.1M to 0.4M daily transactions**, with event-driven spikes
  - TVS often in the **hundreds of millions** range during stronger adoption windows

- **Polygon zkEVM**
  - materially smaller footprint in 2024
  - often **tens of thousands to low hundreds of thousands of transactions per day**
  - TVS generally **well below Era and often below Linea**

These are real-world observed usage ranges, not vendor TPS claims. They also reveal an important truth: **actual bottlenecks in 2024 were usually ecosystem demand, proving cadence, and fee economics, not raw theoretical VM throughput**.

### Finality behavior

What matters operationally is not “TPS max” but:

- soft confirmation latency: usually seconds
- proof finality: often **minutes to hours**, depending on batch cadence and prover backlog
- native L1 withdrawal: generally after proof finalization, so much faster than optimistic rollups’ challenge-window exits

By contrast, optimistic rollups commonly had:
- fast L2 inclusion
- but **~7 day canonical L1 withdrawal latency**

### Cost structure after EIP-4844

After blobs, fee economics improved for all rollups using Ethereum DA:
- DA costs dropped relative to calldata-only models
- batch economics became more favorable
- compression efficiency still mattered
- proving cost remained a differentiator for ZK systems

This especially benefits high-volume systems because fixed proving overhead amortizes better across larger batches.

---

## 8. Shared sequencing, Espresso, Astria

A major recent shift is the recognition that “single sequencer now, decentralize later” creates persistent problems:

- censorship risk
- opaque ordering / MEV capture
- poor cross-rollup composability
- difficult liveness assumptions during operator outages

### Shared sequencing

A **shared sequencer** is a sequencing layer used by multiple rollups. The hoped-for benefits are:

- common ordering guarantees
- reduced trust in one rollup operator
- atomic cross-rollup coordination
- improved interop and potentially better MEV policy design

But it also introduces:
- another layer in the stack
- new liveness dependencies
- potentially new fee markets and governance complexity

### Espresso

**Espresso** has been developing a shared sequencing / confirmation approach meant to give rollups:
- fast confirmations
- shared ordering
- interoperability benefits

The conceptual value for ZK rollups is strong: sequencing can be decentralized independently of proving. A rollup could still settle with validity proofs but outsource ordering.

### Astria

**Astria** pushes a more modular sequencing architecture:
- sequencing separated from execution
- rollups plug into a common ordering layer
- useful for appchains and specialized rollups that do not want to run their own sequencer set

### Relevance to zkSync Era, Linea, Polygon zkEVM

As of the latest public architecture I can responsibly describe, these three are still best understood as **individually operated rollups**, not as fully shared-sequenced systems in production default mode.

For researchers, the interesting trend is:

```text
Today:
  per-rollup sequencer + per-rollup prover + Ethereum settlement

Emerging:
  shared sequencer + per-rollup execution/proving + Ethereum settlement
```

If that model sticks, future differentiation among ZK rollups may shift away from ordering and toward:
- proving cost
- VM equivalence
- compression
- developer tooling
- liquidity/network effects

---

## 9. Why this matters to app developers

## 9.1 Deployment and execution costs

Developers pay for:
- deployment bytecode size
- runtime gas
- calldata / blob-amortized publication
- bridge interactions

Implications by design:

- **zkSync Era**
  - can be attractive on fee efficiency
  - but compatibility differences may require code review and test adaptation

- **Linea**
  - usually lower migration friction from Ethereum
  - especially good if your tooling stack assumes near-standard EVM behavior

- **Polygon zkEVM**
  - also strong portability posture
  - but ecosystem scale and liquidity depth matter as much as architecture

## 9.2 Finality time

Ask three separate questions:

1. **When does the sequencer include my tx?**
   - usually seconds

2. **When is the state posted to Ethereum?**
   - batch dependent

3. **When is it finalized by proof?**
   - minutes to hours in real operation, depending on prover and batching cadence

For many consumer apps, soft confirmations are enough.  
For exchanges, bridges, lending protocols, and intent systems, **proof finality** matters.

## 9.3 Withdrawal latency

This is where ZK rollups clearly outperform optimistic rollups natively.

- **ZK rollup**: withdraw after the relevant state is proven and finalized on L1
- **Optimistic rollup**: withdraw after challenge window, often around a week

Bridges can hide this with fast liquidity exits, but then the app depends on:
- bridge inventory
- relayer trust or credit risk
- extra fees

If your app needs **trust-minimized exits**, the difference is decisive.

## 9.4 Tooling and debugging risk

Type-2-ish systems reduce risk for:
- auditors
- low-level infra teams
- custom tracing/indexing stacks
- existing Solidity codebases

Custom-VM systems may still be excellent, but the engineering burden shifts left:
- more testing
- more chain-specific assumptions
- more careful audit review of edge cases

---

## 10. Bottom line comparison

```text
zkSync Era
  Best read as: ZK-efficiency-forward
  Strengths: strong scaling posture, busy ecosystem, custom-proof-friendly VM
  Tradeoff: less exact EVM equivalence

Linea
  Best read as: compatibility-forward zkEVM
  Strengths: high dev portability, Ethereum-like behavior
  Tradeoff: proving high-fidelity EVM is expensive/complex

Polygon zkEVM
  Best read as: compatibility-focused zkEVM tied to strong proving research lineage
  Strengths: good portability, deep Polygon research bench
  Tradeoff: smaller production footprint than Era in 2024; proving stack nomenclature is easy to oversimplify
```

---

# Decision tree for app deployers

```text
Start
 |
 |-- Do you need trust-minimized native withdrawals faster than ~7 days?
 |       |
 |       |-- Yes -> Prefer ZK rollup
 |       |
 |       |-- No  -> Optimistic rollup remains viable
 |
 |-- Is bytecode-level / near-exact EVM behavior critical?
 |       |
 |       |-- Yes -> Favor Linea or Polygon zkEVM
 |       |
 |       |-- No  -> zkSync Era becomes more attractive
 |
 |-- Do you depend on obscure EVM edge cases, custom tooling, or low-level gas assumptions?
 |       |
 |       |-- Yes -> Favor Type-2-ish zkEVMs first
 |       |
 |       |-- No  -> Custom-VM designs are acceptable
 |
 |-- Is current ecosystem liquidity / usage a top priority?
 |       |
 |       |-- Yes -> Prefer the chain with the strongest observed production demand
 |       |          (through mid-2024, that was generally zkSync Era among these three)
 |       |
 |       |-- No  -> Optimize for technical fit instead
 |
 |-- Is sequencing decentralization a hard requirement today?
 |       |
 |       |-- Yes -> Be cautious: most production rollups still rely on centralized sequencing
 |       |          and shared sequencing is still an active transition area
 |       |
 |       |-- No  -> Current production rollups are usable with known trust tradeoffs
 |
 |-- Recommended bias
         |
         |-- Want highest compatibility, lowest migration friction -> Linea
         |
         |-- Want compatibility plus Polygon ecosystem/proving roadmap -> Polygon zkEVM
         |
         |-- Want aggressive ZK-oriented architecture and can handle VM differences -> zkSync Era
```

## Final take

For protocol researchers, the most important framing is this: the real competition is not “which proof system has the coolest name,” but **which layer of the stack absorbs complexity**.

- **Optimistic rollups** absorb complexity in dispute resolution and delayed exits.
- **Type-2-ish zkEVMs** absorb complexity in proving faithful EVM semantics.
- **Custom-VM ZK rollups** absorb complexity in developer compatibility and migration.

Among the three compared here:

- **zkSync Era** pushes complexity into VM redesign to improve proof economics.
- **Linea** pushes complexity into the prover so developers see something closer to Ethereum.
- **Polygon zkEVM** aims for compatibility while leveraging Polygon’s fast-moving proving research, though one must distinguish production deployment from roadmap branding.

For app teams, the selection rule is simple:
- choose **optimistic** if week-long canonical exits are acceptable and compatibility dominates,
- choose **ZK** if faster trust-minimized finality and exits matter,
- and among ZK choices, decide whether you want **compatibility-first** or **proof-efficiency-first**.

If you want, I can turn this into:
1. a **shorter CTO briefing memo**,  
2. a **research-note style version with citations and footnotes**, or  
3. a **comparison matrix with estimated finality/withdrawal/tooling scores**.
