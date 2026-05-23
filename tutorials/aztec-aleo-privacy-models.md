---
layout: default
title: "Aztec Accounts vs Aleo Records; Privacy Models Deep Dive"
slug: aztec-aleo-privacy-models
ecosystem: cross
description: "Side-by-side: what is public, what is private, where you prove things, and which model fits which workload."
schema: article
og_type: article
---

# Aztec Accounts vs Aleo Records: Privacy Models Deep Dive

Below is a protocol-engineering comparison of **Aztec** and **Aleo** as they actually model private state today: both are hybrids, but they put the “privacy primitive” in very different places.

- **Aztec** is best thought of as **account-based execution with note-based private state**.
- **Aleo** is best thought of as **program execution over records**, where records are **single-use private objects** very close to a UTXO model, plus **public mappings/finalize state** for persistent public state.

That difference drives almost everything else: lifecycle, wallet architecture, proving flow, composability, and privacy leakage.

---

## 1) Conceptual model and why each chose it

### Aztec: Ethereum-style contract semantics, private state as notes

Aztec’s design goal is not “private Bitcoin.” It is **private smart contracts with Ethereum-like programmability and account abstraction**. The protocol therefore keeps an account/public-state worldview for the execution environment, but it does **not** store private balances as mutable slots. Instead, private state is represented as **notes** committed into Merkle trees.

Why?

1. **Mutable private storage is hard to prove privately in a shared state machine.**  
   If every contract had arbitrary encrypted storage cells, the proving and synchronization model becomes much uglier: who can read which cell, how do clients discover their state, how do you update without leaking key paths, etc.

2. **Notes give clean spend semantics.**  
   A note can be committed, later spent by revealing a witness and producing a **nullifier**, exactly once. That maps well to private assets, claims, positions, and rights.

3. **Aztec still wants composable contract calls.**  
   Unlike a pure UTXO chain, Aztec wants contracts to call other contracts in a call stack, split work between **private execution** and **public execution**, and let accounts define custom authorization logic. So it keeps an account-oriented contract model around the note layer.

So Aztec is not “account model instead of notes.” It is **account model for identity/auth/public state, notes for private state**.

### Aleo: records first, programs consume and produce them

Aleo’s private state model starts from a different place: the core object is the **record**. A record is a private, owned data container that is **consumed and recreated** rather than updated in place. That is fundamentally UTXO-like.

Why Aleo chose this:

1. **Single-use private objects simplify proving and parallelization.**  
   Consuming existing records and creating new records is a natural fit for succinct proofs. You prove ownership/spendability of inputs and correctness of outputs. No hidden mutable slot needs to be updated in place.

2. **Ownership is explicit at the data layer.**  
   Each record has an owner and private contents. Wallets scan for records they can decrypt/control, much like note discovery on shielded systems.

3. **Aleo programs are built around transition semantics.**  
   A transition consumes inputs and emits outputs. Records make this model canonical for private execution.

Aleo is also not purely UTXO in the Bitcoin sense, because it has **public state** via `mapping`/`finalize`-style logic. But for private state, the record is the primary abstraction.

### The core philosophical split

- **Aztec:** “Private state should feel like a smart-contract platform with hidden state objects.”
- **Aleo:** “Private state should be first-class owned objects that programs transform.”

That distinction matters when you build higher-level apps:

- Aztec tends to model private assets and app state as **contract-managed note sets**.
- Aleo tends to model them as **records flowing through transitions**.

---

## 2) Note vs. record lifecycle: creation, spending, nullifying

## Aztec note lifecycle

A private Aztec contract typically defines a **note type** for some piece of state: balance note, claim note, position note, receipt note, etc.

### Creation

During private execution, the contract computes a note, hashes it into a **note hash/commitment**, and emits it to be inserted into Aztec’s note tree. Separately, the wallet/PXE handles **note discovery** and encryption/logging so the intended owner can recover the note contents.

At protocol level, what settles publicly is not the plaintext note. What appears is a **commitment** and associated proof artifacts.

Conceptually:

1. User runs private function locally.
2. Function creates one or more notes.
3. The proof attests those notes were created according to contract logic.
4. New note hashes are appended on-chain/in-rollup.

### Discovery

This is one of the underappreciated engineering costs of note systems.

A note is only useful if the recipient can discover it. Aztec uses encrypted logs / tagging / PXE-side scanning machinery so wallets can identify notes meant for them. This is similar in spirit to how shielded systems require recipient-side scanning. It is not “the chain stores your hidden balance in a slot you can query.” The wallet must actually recover notes.

### Spending

To spend a note, the prover supplies:

- the note preimage/witness,
- a Merkle membership proof showing the note commitment exists in the note tree,
- authorization proving the spender is allowed to consume it,
- contract logic showing the spend is valid.

The note itself is not updated. Instead it is **consumed** and replaced by new notes representing the new state, often including change.

### Nullification

A spent note produces a **nullifier**. The nullifier is public and inserted into a nullifier set/tree to prevent double spends.

Important point: in Aztec, the nullifier is the public anti-double-spend marker. The chain does not need to know the note plaintext, only that:
- the note existed,
- this nullifier is valid for it,
- the nullifier has not appeared before.

This makes Aztec private state operationally UTXO-like even though the surrounding execution model is account-based.

---

## Aleo record lifecycle

Aleo records are more directly exposed in the programming model.

### Creation

A transition can output one or more **records**. A record includes an `owner` and other fields. The owner is typically private, and the record is encrypted/committed so only the recipient can use it.

At chain level, Aleo stores enough public data to verify inclusion and later prevent double-spend, while the sensitive payload remains encrypted/private.

### Discovery

Wallets scan transitions/outputs to discover records they can decrypt. This is closer to shielded UTXO systems than to account-based chains. The application developer must assume record fragmentation and client-side state reconstruction as normal behavior.

### Spending

Aleo transitions take records as inputs. If you spend a record, you do not mutate it. You consume it and produce new records as outputs.

Typical private transfer pattern:

- input record: `{ owner: Alice, amount: 10 }`
- outputs:
  - `{ owner: Bob, amount: 4 }`
  - `{ owner: Alice, amount: 6 }`

### Nullification / serial numbers

Aleo prevents double-spending with a serial-number/nullifier style mechanism. In Aleo terminology, a spent record yields a **serial number** derived from spend authority over that record. Once that serial number is on-chain, the record cannot be spent again.

So on the lifecycle mechanics, Aztec notes and Aleo records are very similar:
- both are created as commitments,
- both are discovered off-chain by wallets,
- both are single-use private objects,
- both are spent by witness + inclusion proof,
- both are killed by a public anti-double-spend marker.

The real difference is where the abstraction lives:
- in Aztec, notes are **contract-managed private state objects** inside a broader smart-contract system;
- in Aleo, records are **the primary private data model** of the VM/programming model.

---

## 3) Equivalent code examples

These are intentionally schematic, not copy-paste complete deployments. The point is to show the **shape** of equivalent logic.

### Aztec / Noir: private transfer using notes

```rust
// Schematic Aztec Noir-style code

#[note]
struct BalanceNote {
    owner: AztecAddress,
    amount: u128,
    randomness: Field,
}

#[storage]
struct Storage {
    balances: PrivateSet<BalanceNote>,
}

#[aztec(private)]
fn mint_private(to: AztecAddress, amount: u128) {
    let note = BalanceNote {
        owner: to,
        amount,
        randomness: std::rand::random(),
    };

    storage.balances.insert(note);
}

#[aztec(private)]
fn transfer_private(
    input_note: BalanceNote,
    recipient: AztecAddress,
    send_amount: u128,
) {
    // Ownership check is enforced by note spending/auth logic
    assert(input_note.amount >= send_amount);

    // Consume old note -> emits nullifier
    storage.balances.remove(input_note);

    let change = input_note.amount - send_amount;

    let recipient_note = BalanceNote {
        owner: recipient,
        amount: send_amount,
        randomness: std::rand::random(),
    };
    storage.balances.insert(recipient_note);

    if change > 0 {
        let change_note = BalanceNote {
            owner: input_note.owner,
            amount: change,
            randomness: std::rand::random(),
        };
        storage.balances.insert(change_note);
    }
}
```

What matters here:

- The contract manages a **private set of notes**.
- Spending a note means **removing** it from the private set, which results in a nullifier.
- New balances are represented as **new notes**, not an updated slot.

### Aleo / Leo: private transfer using records

```rust
// Schematic Leo-style code

program token_like.aleo {

    record Balance {
        owner: address,
        amount: u64,
    }

    transition mint_private(to: address, amount: u64) -> Balance {
        return Balance {
            owner: to,
            amount: amount,
        };
    }

    transition transfer_private(
        input: Balance,
        to: address,
        amount: u64
    ) -> (Balance, Balance) {
        assert_eq(input.owner, self.caller);
        assert(input.amount >= amount);

        let recipient = Balance {
            owner: to,
            amount: amount,
        };

        let change = Balance {
            owner: self.caller,
            amount: input.amount - amount,
        };

        return (recipient, change);
    }
}
```

What matters here:

- The **record** is the state object.
- The transition **consumes** one record and **produces** two.
- The serial-number/nullifier behavior is part of the VM/ledger semantics, not something the app manually encodes.

### Equivalent functionality, different mental model

These snippets implement the same transfer pattern, but the developer experience differs:

- In **Aztec**, the transfer is a private contract method operating over a contract-owned note set.
- In **Aleo**, the transfer is a transition over explicit input/output records.

That sounds subtle, but it changes composability and state architecture substantially.

---

## 4) Composability tradeoffs: note hooks vs. program-to-program calls

This is where the two ecosystems diverge hardest.

## Aztec composability

Aztec wants something close to DeFi-style contract composability, but private state makes synchronous composability difficult.

### Strengths

1. **Shared contract world**
   Private and public functions can coexist in one contract system. A protocol can maintain public registries, public accounting, and private user state in one architecture.

2. **Private call stacks**
   Private execution can compose multiple contract calls under a single proof flow.

3. **Rich app-specific note logic**
   Notes are not just currency outputs. They can encode rights, positions, receipts, pending actions, claims. A contract can interpret note creation/spend as a semantic event.

This is the context in which people talk about **note hooks** or note-driven composability: a note is not merely value; it can carry app semantics that downstream contracts or spending logic react to.

### Weaknesses

1. **State discovery burden**
   Every composable private protocol inherits note discovery complexity.

2. **Hidden state is harder to compose synchronously**
   If another protocol needs to reason about your hidden position, it often needs either:
   - a proof about that position,
   - a special interface/note format,
   - or some public side effect.

3. **Cross-contract private reads are not free**
   Composability exists, but the ergonomics are not Ethereum’s “just read another contract storage slot.”

In practice, Aztec composability tends to work best when protocols agree on note formats, auth patterns, and proof boundaries.

## Aleo composability

Aleo’s model is cleaner at the transition level.

### Strengths

1. **Program-to-program calls are explicit**
   You call another program’s function as part of a transition graph. This is clearer than trying to infer effects from hidden note sets.

2. **Records compose naturally as capabilities**
   A record can represent ownership of value, a claim ticket, a permission, a position. Passing it to another program is conceptually simple: consume one record, produce another.

3. **Good fit for object-capability design**
   Since private state is packaged as owned records, many app patterns become “prove you own this object and transform it.”

### Weaknesses

1. **Cross-program statefulness is awkward for rich DeFi**
   If your protocol needs large, continuously mutating shared state, a record-first model becomes cumbersome. You end up either:
   - sharding state into many records, or
   - pushing logic into public mappings/finalize state.

2. **Record fragmentation**
   Like UTXO systems, applications must manage change records and object splitting/merging.

3. **Asynchrony around public state**
   When public mappings/finalize logic are involved, you do not have the same “everything is a synchronously mutable object” feel as an account-based VM.

### Bottom line on composability

- **Aztec** is better aligned with **complex application environments** where private and public logic need to live inside one contract system and feel somewhat Ethereum-like.
- **Aleo** is better aligned with **object-flow computation**, where programs transform private owned records and call one another explicitly.

If you are building a private DEX, lending market, or generalized rollup-native application, Aztec’s model is usually more natural. If you are building private asset flows, ticketing, credentials, games with owned items, or capability-oriented workflows, Aleo’s record model is often cleaner.

---

## 5) Privacy properties: what is hidden, what leaks

Neither system gives “nothing leaks.” They hide contents, not existence.

## Aztec privacy properties

### Hidden

- Note contents
- Private function inputs/locals
- Ownership details tied to notes
- Most private state transitions, except what must be committed publicly

### Public / leaked

- **Nullifiers** for spent notes
- **Note commitments/hashes** for newly created notes
- Transaction inclusion timing and ordering
- Fee payment mechanics
- Any **public function calls** and public storage writes
- The number/pattern of notes consumed and created
- Any L1/L2 messaging side effects

Aztec’s biggest privacy caveat is that a private transaction is still a transaction in a shared sequenced system. Observers can correlate:
- when nullifiers appeared,
- how many notes were created,
- whether a public leg followed a private leg,
- fee patterns.

So Aztec hides **state contents**, but not all **activity metadata**.

## Aleo privacy properties

### Hidden

- Record contents
- Record ownership
- Private transition inputs/outputs
- Private intermediate computation

### Public / leaked

- Program ID / function being called
- Transition timing and ordering
- Number of consumed and produced records
- Serial numbers for spent records
- Public outputs / finalize operations / mapping updates
- Fee source/mode and transaction graph metadata

Aleo’s leakage profile is more obviously UTXO-like:
- records are private,
- spends create serial numbers,
- output count and structure leak,
- change behavior can fingerprint wallets/apps.

### Which leaks less?

It depends on the application.

- For **pure private transfers**, both models can hide contents well while leaking graph metadata.
- For **complex multi-step applications**, Aztec can sometimes hide more of the logical computation inside a private call/proof boundary, but if the flow touches public state, significant structure becomes visible.
- Aleo tends to make the **program/function invocation graph** more explicit, which is good for clarity but leaks more application-shape metadata.

A good engineering summary is:

- **Aztec** optimizes for hiding **private contract state transitions** inside a richer contract environment.
- **Aleo** optimizes for hiding **private owned objects** moving through a program graph.

---

## 6) Real production examples: who uses which model and why

This category is asymmetric today because Aztec’s current programmable private-contract stack is newer than its earlier production system, while Aleo’s ecosystem is centered around its VM/program model.

## Aztec examples

### zk.money / Aztec Connect era

The clearest real production example from Aztec is **zk.money** and the **Aztec Connect** architecture. Users deposited assets into Aztec, received private notes representing balances/claims, and could interact with integrated protocols while keeping note contents hidden.

Why the note model fit:
- user balances were naturally represented as private notes,
- shielding/unshielding maps cleanly to note creation/nullification,
- bridge interactions could aggregate many users while maintaining hidden per-user state.

This was not just a theoretical design; it ran in production and showed the operational realities of note systems:
- note discovery matters,
- nullifier handling matters,
- wallet UX is harder than account-slot UX.

### New Aztec app patterns

On the newer programmable Aztec stack, the most natural applications are:
- private tokens,
- private payments,
- private membership/credential gating,
- private voting,
- hidden positions/claims in DeFi-like protocols.

These use notes because app state is often “a privately owned claim” rather than “a public slot with an encrypted value.”

## Aleo examples

### credits.aleo

The most important production Aleo program is the native **`credits.aleo`** system itself. It demonstrates the record model directly:
- private credits are carried in records,
- transfers consume and create records,
- spending is one-time via serial-number semantics.

Why the record model fits:
- native private asset transfer is the canonical UTXO-like use case,
- ownership and spendability are easy to express,
- wallets can manage records locally.

### ARC-21-style private token programs and private asset apps

Aleo token and asset applications generally mirror the same pattern:
- balances are represented as records,
- transfers are transition-based,
- public admin/config state, when needed, goes into mappings/finalize logic.

Why developers choose this:
- the record abstraction is already the VM’s native private object,
- passing records between users/programs is straightforward,
- many asset apps do not need a large amount of synchronously shared hidden state.

### Wallet/payment-oriented Aleo apps

Aleo’s strongest real-world use cases so far have clustered around:
- private transfers,
- private token movement,
- owned objects/collectibles,
- simple private game assets or credentials.

That is exactly where a record model shines. It is less ideal for high-frequency DeFi state machines that want shared, continuously evolving private state across many protocols.

---

## Final comparison

The simplistic story is “Aztec is account-based, Aleo is UTXO-like.” That is directionally true, but technically incomplete.

The more accurate statement is:

- **Aztec** is a **smart-contract platform with account abstraction and public state**, where **private state is represented as notes**. Its privacy model is built around contract-managed note commitments and nullifiers.
- **Aleo** is a **program VM where private state is primarily records**, consumed and recreated across transitions, with public mappings/finalize state for shared public persistence.

That leads to different strengths:

### Choose Aztec when you want:
- Ethereum-like contract architecture
- private + public logic in one application
- sophisticated cross-contract application design
- app-specific private state objects managed by contracts

### Choose Aleo when you want:
- object-style private state
- clean owned-asset flows
- explicit transition semantics
- simpler mental mapping from “private thing I own” to “record I spend”

And different costs:

### Aztec costs
- heavier note discovery and wallet/PXE complexity
- more complex private composability engineering
- metadata leakage around note/nullifier patterns and public legs

### Aleo costs
- UTXO-like fragmentation
- awkwardness for large shared hidden state
- application graph leakage through program/transition structure

If you are engineering privacy-preserving applications, the practical distinction is this:

- In **Aztec**, you usually ask:  
  **“What notes does this contract own/manage, and how do private and public execution interact?”**

- In **Aleo**, you usually ask:  
  **“What records are consumed and produced, and which public finalize state, if any, must this transition update?”**

That is the real privacy-model difference. It is not just syntax. It is the difference between a **private contract state machine** and a **private object-flow machine**.

<!-- cta-block:start -->

---

## Where to go next

Thanks for reading this far. If "Aztec vs Aleo privacy models" connected with where you are, three concrete next steps:

### Learn more in Aztec

The full [Midnight ZK Cookbook index](https://battam1111.github.io/midnight-zk-cookbook/) has 17 tutorials across Midnight, Aleo, Aztec, Noir, and risc0 plus 4 Chinese translations. Adjacent tutorials are listed by ecosystem on that page.

### Find paid work in Aztec

Bounty Radar tracks open ZK bounties across Algora, GitHub labels, Drips Wave, Code4rena, and Bountycaster. Browse the [Aztec sub-feed](https://battam1111.github.io/bounty-radar-data/widget.html?ecosystem=aztec); JSON at [`/aztec.json`](https://battam1111.github.io/bounty-radar-data/aztec.json). The free tier is poll-based; the [$19/mo Hobbyist tier](https://polar.sh/checkout/polar_c_BbZbN6eJnZ7rwsUfT1pMsj4lTftwnfMoGdWBo0KozKU) pushes one filter to your Telegram in real time.

### Audit your own ZK pipeline

[zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor) is the free MIT-licensed CLI that scores any ZK project on tests, CI, docs, security, reproducibility, and language toolchain (supports Compact, Leo, Noir, Cairo, and 7 Rust zkVMs). Drop it into a GitHub Action with [zk-doctor-action](https://github.com/Battam1111/zk-doctor-action) for diff-aware PR comments. The [$15/mo Pro tier](https://github.com/Battam1111/zk-pipeline-doctor#pro-tier) adds four cross-ecosystem deep detectors (circuit complexity, proving-system pitfalls, verifier soundness, multi-file consistency).

---

*Drafted with AI assistance and reviewed by the author before publishing. See [DISCLOSURE](https://battam1111.github.io/midnight-zk-cookbook/DISCLOSURE.html) for the full process.*

<!-- cta-block:end -->
