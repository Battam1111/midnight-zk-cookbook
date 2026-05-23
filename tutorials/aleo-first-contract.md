---
layout: default
title: "Your First Aleo Contract in Leo: Mappings, Records, and Transitions"
slug: aleo-first-contract
ecosystem: aleo
description: "Bottom-up tour of Aleo Leo: build a working program with records, mappings, and transitions, then deploy it locally."
schema: article
og_type: article
---

# Your First Aleo Contract in Leo: Mappings, Records, and Transitions

## TL;DR

This tutorial is written against the Aleo reference primer fetched on **2026-05-20** and the **`mainnet` snarkVM README**. The Leo documentation URLs in the primer resolve to “Page Not Found,” so this guide is intentionally strict about what is and is not verifiable.

What is safe to say:

- Aleo programs are built around **provable execution**.
- The core concepts you need for a first contract are **transitions**, **mappings**, and **records**.
- A practical first design is:
  - use a **mapping** for shared on-chain state,
  - use a **record** for privately owned state,
  - expose behavior through **transitions** that update one or both.

What is **not** safe to do from the provided primer:

- Paste Leo syntax and claim it is current.
- Invent exact compiler commands, decorators, storage declarations, or transition signatures.
- Claim a specific Leo language version.

So this is a production-honest tutorial: it explains how to design your first Aleo contract correctly, what invariants matter, and where you must **verify against your local installation/compiler version** before shipping.

---

## What version this tutorial is written against, and why that matters

The first thing a working developer should know is whether the examples in a tutorial can be copied verbatim. In this case, the answer is **no**, and that is not a defect of Aleo so much as a limitation of the source material provided here.

The verified references available in the primer are:

- Aleo Leo syntax page: unavailable at the fetched URL
- Aleo Leo language page: unavailable at the fetched URL
- snarkVM README on the `mainnet` branch: available

From the snarkVM README, the following are verifiable:

- snarkVM exists as the VM and proving stack around Aleo.
- It is primarily used as a Rust library.
- It can be built from source.
- The project is maintained in the `ProvableHQ/snarkVM` repository.
- Rust installation and a basic dependency declaration are documented.

Verifiable snippets from the README include:

```toml
[dependencies]
snarkvm = "major.minor.patch"
```

and:

```bash
git clone --branch staging --single-branch https://github.com/ProvableHQ/snarkVM.git
cd snarkVM
cargo build --release
```

That is useful for environment context, but it does **not** give us authoritative Leo program syntax. Because of that, this tutorial does **not** present copy-paste Leo code. Instead, it teaches the contract design you should implement once you check the exact syntax in your local toolchain.

That may sound conservative, but for zero-knowledge systems it is the correct posture. Small syntax differences can reflect larger semantic differences:

- whether a value is public or private by default,
- how records are created or consumed,
- how mappings are keyed,
- whether a transition can emit or return records,
- how ownership is represented,
- and which operations are allowed inside provable execution.

In other words: if you are evaluating whether to ship on Aleo, the right first step is not memorizing syntax. It is understanding the **state model**.

For the rest of this tutorial, I will describe the three concepts in a way that remains valid even if the surface syntax changes:

- **Mappings**: shared program state indexed by keys.
- **Records**: privately owned data objects.
- **Transitions**: provable state transitions, the callable units of your program.

That mental model is the durable part.

---

## The Aleo mental model: transitions, mappings, and records

If you come from EVM development, it is tempting to look for direct analogies:

- mappings ≈ storage maps,
- records ≈ structs or UTXO-like notes,
- transitions ≈ functions.

Those analogies are helpful, but incomplete. Aleo’s design is shaped by zero-knowledge proving and private ownership.

### Transitions

A **transition** is the executable action in your program. Conceptually, a transition:

1. takes some inputs,
2. proves that the requested state change is valid,
3. updates public/shared state and/or private state,
4. emits outputs that can be consumed later.

The important shift is that a transition is not just “run some code.” It is “run code in a way that can be **proven**.” That means every branch, arithmetic constraint, ownership check, and state dependency should be designed with proving costs and privacy boundaries in mind.

For a first contract, think of transitions like:

- `initialize`
- `mint`
- `transfer`
- `redeem`
- `set_admin`
- `claim`

The exact names and syntax are compiler-specific. The design principle is not.

### Mappings

A **mapping** is the part of your program state that is naturally shared and addressable by key. This is where you keep data that many users or observers may need to reference.

Common uses:

- an admin key or address,
- a counter,
- a registry of authorized IDs,
- nullifiers or spent flags,
- publicly queryable metadata,
- aggregate balances or supply numbers.

Mappings are appropriate when the system needs a canonical answer to “what is the current value for key K?”

For your first Aleo contract, a mapping is usually the easiest place to put minimal global state. Examples at the design level:

- `admins[account] = true/false`
- `allowlist[user] = true/false`
- `spent[id] = true`
- `total_supply[token_id] = n`

Again, these are conceptual examples, not guaranteed Leo syntax.

### Records

A **record** is the Aleo concept most developers need to pause on. The easiest way to think about a record is:

> a privately owned state object that can carry data and is intended to move between users or be consumed by later transitions.

If you know UTXO systems or note-based privacy systems, that is a closer analogy than account storage.

Records are useful when you want a user to hold data privately, such as:

- a private balance note,
- a claim ticket,
- a membership credential,
- a coupon or voucher,
- a receipt proving entitlement to a later action.

A record generally answers the question: “what private object does this user control that allows them to perform the next step?”

### Why Aleo needs both mappings and records

Aleo is most interesting when you use both.

Mappings are good for:

- coordination,
- canonical state,
- anti-replay flags,
- public policy.

Records are good for:

- private ownership,
- selective disclosure through proofs,
- transferring rights without exposing all state.

Transitions connect them:

- a transition may read a mapping to verify policy,
- consume a record to prove ownership,
- write a mapping to prevent replay,
- and create a new record for the next owner.

That pattern is the heart of many Aleo applications.

---

## A concrete first contract design: private vouchers with public spend protection

Because we cannot verify current Leo syntax from the primer, the best way to learn is through a contract design that is simple but realistic. A good first contract is a **private voucher system**.

Why this example works:

- it needs a **mapping** for public anti-double-spend protection,
- it needs a **record** to represent a privately held voucher,
- it needs several **transitions**,
- and it forces you to think about privacy boundaries and invariants.

### Goal

We want a program that lets an authorized issuer create vouchers. A user can privately hold a voucher and later redeem it. The chain should prevent the same voucher from being redeemed twice.

### State model

Conceptually, the program has:

**Shared state in mappings**
- an issuer/admin registry,
- a mapping of redeemed voucher IDs.

**Private state in records**
- a voucher record owned by a user, carrying:
  - owner,
  - voucher ID,
  - amount or entitlement,
  - optional metadata.

### Transition set

At minimum, the contract design needs three transitions.

#### 1. Initialize or configure issuer authority

Purpose:
- establish who can issue vouchers.

Conceptually:
- set the program’s issuer/admin state in a mapping.

Things to verify in your local version:
- whether initialization is a special deployment-time action or an ordinary transition,
- how access control is encoded,
- whether mapping writes require prior existence checks.

#### 2. Issue voucher

Purpose:
- authorize creation of a private voucher record.

Inputs at the design level:
- issuer authority,
- recipient identity,
- voucher ID,
- amount or entitlement.

Behavior:
- verify caller or provided authority is authorized,
- verify voucher ID has not already been issued if uniqueness matters,
- create a voucher record owned by the recipient.

Questions you must resolve locally:
- whether the issuance uniqueness check should live in a mapping,
- whether the record itself can encode enough entropy/uniqueness,
- whether the program should also maintain public aggregate totals.

#### 3. Redeem voucher

Purpose:
- consume a private voucher and prevent replay.

Inputs:
- the voucher record,
- any public claim context needed by the application.

Behavior:
- prove ownership/control of the voucher record,
- verify the voucher has not been redeemed,
- mark the voucher ID as redeemed in a mapping,
- optionally produce another record, a receipt, or some public output.

This transition demonstrates the classic Aleo composition:

- private input: the voucher record,
- public/shared consistency: redeemed mapping,
- state change: mark as spent,
- optional output: receipt or follow-on entitlement.

### Data model decisions

Even before syntax, you need to decide what lives where.

#### Put in a mapping only what must be shared

Every shared value increases your public or globally coordinated state footprint. So keep mappings minimal.

Good mapping candidates:
- `authorized_issuer -> bool`
- `redeemed_voucher_id -> bool`

Bad mapping candidates for a privacy-first design:
- per-user full voucher histories,
- sensitive metadata that could remain in records,
- anything that leaks unnecessary linkability.

#### Put in records what belongs to the user

A record should carry the data that the user must later prove control over.

Good record fields:
- owner identity,
- voucher ID,
- amount,
- expiry,
- issuer tag if needed,
- application-specific metadata.

Be careful with fields that create unwanted correlation. If every voucher includes a public-looking sequential ID that later becomes visible during redemption, then privacy can degrade. You should think through whether the voucher ID is:

- globally unique and public,
- unique but hidden until spend,
- or derived in a way that balances replay protection with privacy.

### Invariants to define before implementation

Aleo contracts become safer when you state the invariants first. For this voucher example:

1. **Only authorized issuers can create vouchers.**
2. **A voucher can be redeemed at most once.**
3. **Only the record owner can redeem a voucher.**
4. **Redemption preserves the intended amount or entitlement.**
5. **No transition can forge issuer authority or bypass the redeemed mapping check.**

These invariants are more important than syntax. If your eventual Leo code does not clearly enforce them, the design is incomplete.

### A pseudo-interface, intentionally not Leo syntax

Because the primer does not verify current Leo syntax, here is a conceptual interface only:

- `initialize(admin)`
- `authorize_issuer(issuer)`
- `issue_voucher(recipient, voucher_id, amount) -> voucher_record`
- `redeem_voucher(voucher_record) -> receipt_or_effect`

Treat this as a design checklist, not code.

When you translate it to Leo, verify locally:

- how transitions are declared,
- how records are defined,
- how mappings are declared and read/written,
- how ownership fields are represented,
- how public vs private inputs are annotated,
- and how outputs are returned.

---

## How to reason about privacy and correctness before you write Leo

Aleo development gets easier when you separate two concerns:

1. **correctness**: can invalid state transitions be ruled out?
2. **privacy**: does the contract reveal only what it must?

A first contract often fails not because the arithmetic is hard, but because the privacy boundary is drawn poorly.

### Correctness in a ZK setting

In a normal smart contract, correctness is “the VM executed the checks.” In a ZK system, correctness is “the transition’s constraints and state rules make invalid execution unprovable or unacceptable.”

For the voucher design, correctness means:

- a non-issuer cannot issue,
- a voucher cannot be redeemed twice,
- a user cannot redeem someone else’s voucher,
- redeemed state is updated consistently.

This is why mappings matter. A record alone may represent private ownership, but without some shared anti-replay mechanism, the system can be vulnerable to double use of the same entitlement. The shared mapping acts like the canonical memory of what has already been consumed.

### Privacy tradeoffs

There is no such thing as “fully private by default” without design choices. Even with records, your transition shape may leak more than you intend.

For example:

- If voucher IDs are public at issue time and again at redemption time, observers can correlate events.
- If amounts are always exposed publicly during redemption, then the record is private only in transit, not in value.
- If the redeemed mapping is keyed by something directly linkable to issuance, you may leak user behavior.

A practical design question is:

> what is the minimum public information needed to prevent replay and enforce policy?

Sometimes a public redeemed flag keyed by a hidden commitment-derived identifier is enough. Sometimes your application needs public IDs for business reasons. The right answer depends on the product, but the contract should make the leak explicit.

### Access control deserves extra care

Aleo developers sometimes focus on private ownership and forget that **issuer/admin authority** is equally critical.

For the voucher contract, you need to decide:

- Is there a single issuer?
- Can issuers be added or removed?
- Does every issuance transition directly prove issuer authority?
- Is authority stored in a mapping?
- Is authority itself private or public?

In many first contracts, simple is better:

- one admin,
- one authorized issuer set,
- explicit checks in every privileged transition.

Do not optimize authority management until the basic invariant is working.

### Plan for lifecycle, not just one transition

A record is almost never useful in isolation. Its value comes from lifecycle.

For a first Aleo contract, write down the lifecycle in plain language:

1. Admin configures issuer authority.
2. Issuer creates a voucher record for a user.
3. User stores that record locally.
4. User later submits a redemption transition.
5. The program marks the voucher as redeemed.
6. The voucher record is consumed and cannot be reused.

If you cannot explain the lifecycle without syntax, you are not ready to implement it with syntax.

### Think about failure cases

A production-minded design also asks:

- What happens if a user loses the record?
- Can an issuer revoke or replace a voucher?
- Can a voucher expire?
- What if two redemptions race?
- Is there an audit trail for issued totals versus redeemed totals?

Some of these are application-level choices, not mandatory protocol features. But if you are evaluating whether to ship, they determine whether Aleo’s record model matches your product requirements.

---

## Translating the design into Leo safely

This is the part where many tutorials would dump code. I will not do that here because the Leo syntax pages in the primer are unavailable. Instead, I will show you exactly what to verify when you open your local Leo installation or compiler docs.

### Step 1: Verify how a program declares records

You need to confirm:

- the exact keyword or declaration form for a record,
- how an owner field is represented,
- which field types are allowed,
- whether record fields are private, public, or mixed by default.

Your first implementation should define one small record type only. Keep it minimal:
- owner,
- voucher ID,
- amount.

Add optional metadata later.

### Step 2: Verify how a program declares mappings

You need to confirm:

- the exact syntax for mapping declarations,
- supported key and value types,
- read semantics,
- write/update semantics,
- default behavior for absent keys.

For the voucher example, a single redeemed mapping is enough to learn the model. Avoid multiple mappings until you are clear on initialization and updates.

### Step 3: Verify transition declaration and I/O rules

You need to check:

- how transitions are declared,
- how public and private inputs are marked,
- whether records are passed by value, reference, or consumption semantics,
- whether a transition can return a record,
- whether multiple outputs are supported.

For a first contract, you want one transition that returns a record and one that consumes it. That gives you the full lifecycle.

### Step 4: Verify state access restrictions inside transitions

This is where compiler-version differences matter most.

You must confirm locally:

- how mappings are read inside a transition,
- how absence/presence checks work,
- whether conditional updates are allowed directly,
- whether there are restrictions on loops or dynamic behavior,
- what operations are circuit-friendly and accepted by the compiler.

Do not assume a familiar imperative programming model. Provable languages often restrict control flow, mutation patterns, or built-in operations.

### Step 5: Verify deployment, proving, and execution commands

The primer does not provide Leo CLI commands. So you must **verify against your local installation/compiler version** for:

- project initialization,
- build/compile,
- local execution,
- proof generation,
- test execution,
- deployment submission.

What *is* verifiable from the available primer is that the broader Aleo stack uses snarkVM and that Rust-based tooling exists in the ecosystem. If you need to work directly with the VM or inspect implementation details, the confirmed repository is:

- `https://github.com/ProvableHQ/snarkVM`

### Step 6: Start with the smallest test matrix

Once you have syntax verified locally, your first tests should be behavioral, not exhaustive.

For the voucher contract, test at least:

1. authorized issuer can issue;
2. unauthorized issuer cannot issue;
3. owner can redeem;
4. non-owner cannot redeem;
5. same voucher cannot redeem twice;
6. redeemed mapping changes as expected.

If the local toolchain supports property-style or fuzz-style testing, use it later. For the first pass, deterministic scenario tests are enough.

---

## A practical development workflow using verified pieces of the Aleo stack

Even though the Leo docs in the primer are unavailable, you can still set up a disciplined workflow around the parts we can verify.

### Use snarkVM as your low-level anchor

The primer confirms that snarkVM is the VM and proving library stack, and that it can be used as a Rust dependency.

Documented dependency form:

```toml
[dependencies]
snarkvm = "major.minor.patch"
```

Do not hardcode a version from a random blog post. Pick the published version appropriate for your environment and confirm compatibility with your Leo/compiler setup.

### Build from source when you need implementation clarity

The verified README documents:

```bash
git clone --branch staging --single-branch https://github.com/ProvableHQ/snarkVM.git
cd snarkVM
cargo build --release
```

That does not replace Leo docs, but it does give you an authoritative codebase when you need to inspect current implementation details, benchmarks, or VM-side behavior.

One practical use of this approach is version triage:

- if local Leo behavior differs from an older example,
- and your project depends on a specific network/toolchain combination,
- inspect the corresponding repos and release notes locally before committing to an API shape.

### Keep your contract design document next to your code

This matters more in ZK than in ordinary app code.

Before writing the first transition, write a short design document with:

- state objects,
- which fields are public/shared versus private,
- transition preconditions,
- transition postconditions,
- replay protection strategy,
- authority model,
- expected failure cases.

For the voucher example, the design doc might be one page. That page will save you more time than a copied code sample of uncertain version.

### Treat proving costs as a real product constraint

Aleo programs are not just logic; they are logic that must be proved. So when you move from concept to implementation:

- keep the first transition set small,
- prefer straightforward checks over elaborate abstractions,
- avoid unnecessary per-transition state writes,
- and measure costs in your actual toolchain.

Because the primer does not give benchmark commands or constraint inspection APIs, I will not invent them here. But the engineering principle is clear: **a contract that is elegant on paper but too expensive to prove is not production-ready**.

### Decide early what must be on-chain state

Developers often overuse mappings in their first privacy-preserving contract because mappings feel familiar. In Aleo, ask instead:

- Does this value need global consensus visibility?
- Or can it remain in a user-held record until consumption?

For many applications, the best Aleo design is:
- tiny public/shared state,
- richer private records,
- transitions that reveal only enough to enforce the rules.

That is where Aleo’s model is strongest.

---

## Caveats

1. **No Leo syntax is presented as authoritative here.**  
   The primer’s Leo syntax and language URLs were unavailable at fetch time, so you should **verify against your local installation/compiler version** for all concrete declarations, transition signatures, storage syntax, and CLI commands.

2. **“Contract” is used informally.**  
   In Aleo/Leo contexts, the exact terminology in current docs may emphasize “programs” and “transitions.” Verify the current nomenclature in your toolchain and docs.

3. **Mappings, records, and transitions are explained at the concept level.**  
   The design guidance here is robust, but implementation details such as ownership encoding, visibility modifiers, or return semantics may differ by compiler version.

4. **Privacy is application-specific.**  
   Using records does not automatically guarantee that your application leaks nothing meaningful. Identifier choice, mapping keys, returned outputs, and transition structure can all affect linkability.

5. **Performance claims are intentionally omitted.**  
   The primer does not include verified benchmarking workflows or current proof-cost guidance, so you should measure with your actual local stack.

6. **Network and tooling compatibility must be checked locally.**  
   If you are targeting a specific Aleo network environment, confirm compatibility among your Leo compiler, snarkVM version, and deployment target before building product assumptions on top of examples from any third-party source.

---

## See also

Verified URLs from the provided primer:

- snarkVM README (`mainnet` branch):  
  https://github.com/ProvableHQ/snarkVM/blob/mainnet/README.md

- snarkVM repository:  
  https://github.com/ProvableHQ/snarkVM

- Rust installation via rustup, as referenced by snarkVM build docs:  
  https://rustup.rs/

Primer URLs that were fetched but unavailable at the time:

- Aleo Leo syntax page:  
  https://developer.aleo.org/leo/syntax

- Aleo Leo language page:  
  https://developer.aleo.org/leo/language

If you are implementing this tutorial for real, the next correct step is simple: open your local Leo compiler/docs, verify the exact syntax for **record**, **mapping**, and **transition** declarations, and then implement the voucher design above with the five invariants explicitly tested. That will teach you more about whether Aleo fits your product than any copy-paste sample of uncertain version.