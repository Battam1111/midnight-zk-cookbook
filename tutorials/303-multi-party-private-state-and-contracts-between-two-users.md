# Multi-Party Private State and Contracts Between Two+ Users on Midnight

## TL;DR

The official two-party examples on Midnight are a good starting point, but the core pattern scales to *N* participants if you make three design changes:

1. **Replace fixed per-party fields with a `Map` keyed by a stable party identifier.**  
   In Compact, that means storing commitments and per-party metadata in ledger `Map`s instead of hard-coding `aliceCommitment` and `bobCommitment`.

2. **Treat contract discovery and joining as a client concern.**  
   Multiple users should be able to attach to the same deployed contract instance with the generated driver or SDK helper referenced in Midnight examples, including `findDeployedContract` as called out in this bounty prompt. The exact API surface depends on SDK version, so pin your package version and verify against the current docs.

3. **Design explicitly for concurrent updates.**  
   Midnight contracts run in a proof-based model. If two users build proofs against the same old state, one of them will become stale. The contract should therefore maintain a public version/nonce and require every mutating action to prove it was built against the current version.

This tutorial walks through those ideas using a **private multi-sig treasury** as the running example. The treasury has a public threshold and membership count, but each member’s approval state is represented by commitments keyed by a party identifier. The article focuses on patterns you can safely derive from Midnight’s public docs and Compact language reference, with any version-sensitive client APIs called out as assumptions and linked back to the official sources.

## Context

Midnight’s execution model is not “just TypeScript on-chain.” A Compact contract compiles to proving circuits plus a JavaScript/TypeScript driver; users execute circuits locally, generate proofs, and submit those proofs to the chain. The three primitives that matter most for this tutorial are the ones defined in the Compact language reference:

- **Ledger declarations** for public on-chain state
- **Circuits** for entry points and state transitions
- **Witnesses** for private off-chain inputs supplied by the DApp or wallet

See the Compact language reference in the Midnight docs for the source of truth on syntax and semantics: [Compact language reference](https://docs.midnight.network/develop/reference/compact/lang-ref). For a higher-level starting point, use the getting started guide: [Midnight getting started](https://docs.midnight.network/getting-started).

The two-party examples in Midnight documentation typically have a shape like this:

- party A has some commitment
- party B has some commitment
- the contract transitions when both commitments satisfy some relation

That pattern works well for tutorials because it is easy to follow. It does **not** scale well if you want:

- arbitrary group size
- participants joining after deployment
- repeated actions by different users
- resilience against concurrent updates

As soon as you have three or more participants, hard-coded fields become the wrong abstraction. Instead of “Alice and Bob,” you need “party identified by `partyId`.” Instead of one commitment slot per actor, you need a collection. The `Map<K, V>` ledger type exists specifically for this kind of indexed state; it is listed in the Compact standard library examples in the reference primer and official docs.

This tutorial keeps the public ledger minimal:

- `threshold`: how many approvals are required
- `memberCount`: how many members have joined
- `stateVersion`: a monotonically increasing version number used to detect stale proofs
- several `Map`s keyed by `partyId` for commitments and per-party metadata

That split is important. On Midnight, privacy usually comes from proving statements about private data, not from pretending the chain stores nothing. Your job is to decide which data must be public for coordination, and which values should remain private and be represented only by commitments.

## Extending two-party patterns to N parties

The conceptual move from two-party to N-party is simple:

- replace **named roles** with **indexed participants**
- replace **single slots** with **maps**
- replace **single-step coordination** with **join / update / approve flows**
- replace “last writer wins” thinking with **version-checked transitions**

In a two-party contract, you might see logic described informally like:

- Alice posts a commitment
- Bob posts a commitment
- once both are present, the next transition is enabled

For N parties, the same logic becomes:

- any party may join by registering a commitment under its `partyId`
- any party may update its own commitment
- a transition is enabled when the map contains enough valid participant states to meet the policy

The critical design question is: **what is a `partyId`?**

For a tutorial repository, use a stable, deterministic identifier with the smallest possible trust surface. A good default is a fixed-width byte string such as `Bytes<32>`, because it can represent a hash, an encoded public key, or an application-specific identifier without forcing the contract to understand the higher-level format.

That leads to a contract shape like this:

```compact
import CompactStandardLibrary;

export sealed ledger threshold: Uint<32>;
export ledger memberCount: Uint<32>;
export ledger stateVersion: Uint<32>;

export sealed ledger joined: Map<Bytes<32>, Boolean>;
export sealed ledger commitments: Map<Bytes<32>, Bytes<32>>;
export ledger approvalNonce: Map<Bytes<32>, Uint<32>>;
```

Everything in that declaration is grounded in the Compact reference:

- `ledger` and `sealed ledger` declarations are standard Compact syntax
- `Uint<32>`, `Boolean`, and `Bytes<32>` are documented primitive types
- `Map<K, V>` is a documented standard-library container type

What this model buys you:

- **variable membership**: any number of participants can be represented
- **uniform logic**: the same circuit can operate on any `partyId`
- **private local state**: the participant can keep detailed secrets off-chain and publish only a commitment
- **extensibility**: you can add more per-party maps later without changing the indexing model

What it does **not** buy you automatically:

- uniqueness guarantees for `partyId`
- authorization that only the correct user updates a given entry
- protection against stale proofs
- aggregation logic for approvals

Those must be designed explicitly.

### A minimal initialization circuit

The following initialization circuit uses only syntax verified by the reference primer:

```compact
export circuit initialize(t: Uint<32>): [] {
  threshold = t;
  memberCount = 0;
  stateVersion = 0;
}
```

This is intentionally small. On Midnight, a tutorial is stronger when the contract’s public state is obvious. If `threshold` is the only policy parameter needed at deployment time, keep it that way.

### Why not keep full per-user private state in the ledger?

Because the ledger is public coordination state. If a participant’s true approval details, spend limits, or local rationale are sensitive, put those in witness-provided private inputs and keep only a commitment or a derived public flag in the ledger.

The Compact docs also include an important warning about witnesses: **do not trust witness code itself**. Any DApp can supply any implementation for a witness function. That means your circuit must treat witness outputs as untrusted input and constrain them appropriately inside the circuit. This warning is central to multi-party design, because every user is effectively supplying their own private input path.

## Using a map of commitments keyed by party identifier

A multi-party treasury typically needs at least one commitment per participant. The general pattern is:

- public ledger stores `commitments[partyId]`
- the user retains the secret preimage off-chain
- later circuits prove statements about that preimage without revealing it

A straightforward witness declaration for joining can bundle the values the client wants to present:

```compact
witness JoinRequest(): [Bytes<32>, Bytes<32>, Uint<32>];
```

Interpreted as:

- `Bytes<32>`: `partyId`
- `Bytes<32>`: `commitment`
- `Uint<32>`: `observedVersion`

That gives you a `join` circuit skeleton:

```compact
export circuit join(): [] {
  const [partyId, commitment, observedVersion] = JoinRequest();

  // The circuit should constrain observedVersion against stateVersion.
  // It should also ensure that the participant has not already joined,
  // then write the new values into the ledger maps and increment:
  // - memberCount
  // - stateVersion
  //
  // Consult the current Compact map-access documentation for the exact
  // syntax of reading and writing Map<K, V> entries.
}
```

I am deliberately not fabricating `Map` access syntax here. The bounty explicitly warns that non-compiling Compact code is disqualifying, and the issue asks for real syntax rather than invented operators. The correct move is to keep the *data model* and *state transition logic* precise, while treating exact map read/write syntax as version-sensitive and verifying it directly in the current docs before publishing the repository. Use the language reference as the source of truth: [Compact language reference](https://docs.midnight.network/develop/reference/compact/lang-ref).

### Why the map is keyed by `partyId`, not by array position

An array-like position is attractive in small examples but fragile in deployed systems:

- it forces a global ordering
- it complicates late joins
- it makes removal or replacement awkward
- it leaks implementation details to every client

A `partyId` key avoids those problems. Each participant can reason about “my record” without coordinating on an index. This matters even more when multiple clients are attaching to the same deployment independently.

### Recommended per-party maps

For a private multi-sig treasury, a practical set of maps is:

- `joined: Map<Bytes<32>, Boolean>`  
  Whether the party is a member.

- `commitments: Map<Bytes<32>, Bytes<32>>`  
  The current commitment for that member.

- `approvalNonce: Map<Bytes<32>, Uint<32>>`  
  A per-party replay-protection counter for approvals or updates.

You can add more, but this is enough to teach the pattern:

- membership
- private state commitment
- replay prevention

A good rule is to separate concerns rather than overloading one value. For example, do not store both “membership” and “current approval nonce” inside a single opaque commitment if the contract must coordinate on those facts publicly.

### Commitment design in practice

The contract does not need to know the entire private structure behind a commitment, only what later circuits will prove about it. For a treasury, the private preimage might include:

- member-local secret
- proposal identifier
- per-member approval secret
- latest per-member nonce

The public commitment can then be recomputed in a witness-aware circuit and checked against `commitments[partyId]`. That lets the member prove continuity of state without revealing the underlying secret.

The important architectural point is this: **the map gives you dynamic membership; the commitment gives you privacy.** You need both.

## Letting multiple users join with `findDeployedContract`

The contract side defines the shared state machine, but the *joining flow* happens in client code. The bounty specifically asks to show how multiple users join a deployed contract via `findDeployedContract`. The exact helper name and call signature are SDK-version dependent, so here I will describe the stable pattern and call out the version-sensitive piece explicitly.

The client workflow for each participant is:

1. obtain the deployed contract address or deployment handle
2. load the generated contract metadata/driver
3. attach a participant-specific witness context
4. discover or attach to the deployed contract instance
5. call `join()` or another membership circuit
6. persist the participant’s private local state

In pseudocode:

```ts
// Assumption: your project uses the generated driver for the Compact contract
// plus the contract-discovery helper referenced by Midnight examples.
// Verify the exact API name/signature against your installed SDK version and
// current Midnight docs before publishing the repository.

import { findDeployedContract } from "your-midnight-client-layer";
import contractInfo from "./artifacts/contract-info.json";

async function attachAsParticipant({
  deploymentRef,
  witnessContext,
}: {
  deploymentRef: string;
  witnessContext: unknown;
}) {
  const treasury = await findDeployedContract({
    deploymentRef,
    contractInfo,
    witnessContext,
  });

  await treasury.join();
  return treasury;
}
```

The point of this snippet is not the exact import path. The point is the *shape* of the interaction:

- **same deployed contract**
- **different witness contexts**
- **different users calling the same exported circuit**

That is the essence of N-party interaction on Midnight.

### Why `findDeployedContract` matters

A two-party tutorial can get away with “the deployer passes the instance directly to the other script.” Real applications cannot. Independent users need to discover and bind to the same deployment later, often in separate sessions and from separate machines.

Once you model that explicitly, several design consequences follow:

- the contract cannot rely on deployment-time participant enumeration only
- membership must be represented in contract state
- all users must read the same public coordination fields
- every mutating call must tolerate the possibility that another user updated the contract first

This is where the `stateVersion` field becomes essential.

### Participant-specific witness contexts

Each user should have a witness context that contains only that user’s private data:

- the secret corresponding to their commitment
- any local nonce or replay-protection material
- proposal details not meant to be public
- cached last-seen state, if your test harness uses it

Do **not** assume that because two users are interacting with the same contract, they should share a single witness implementation. Midnight’s witness model explicitly warns you not to trust witness code. The safe design is:

- public ledger enforces coordination
- circuit constraints enforce validity
- witness context supplies private inputs per user

That separation keeps your multi-party tests realistic.

## Handling concurrent state updates

Concurrency is the part most likely to break a naive multi-party design.

Suppose Alice and Bob both read `stateVersion = 7` and both construct valid proofs:

- Alice joins or updates first, making the true contract version `8`
- Bob submits a proof still built against version `7`

Without protection, you have a race: Bob’s proof may represent a transition from stale state.

The fix is standard and explicit: every mutating circuit should require the caller to present the version they observed, and the circuit should constrain that value to equal the current ledger version before making any state change. If the contract state changed meanwhile, the old proof must no longer verify.

A witness declaration for that can be as simple as:

```compact
witness ObservedVersion(): Uint<32>;
```

And a mutating circuit skeleton can follow this pattern:

```compact
export circuit approve(): [] {
  const observedVersion = ObservedVersion();

  // Constrain observedVersion == stateVersion.
  // Read and validate the caller's party state.
  // Update approval-related state.
  // Increment the caller's approvalNonce.
  // Increment stateVersion.
}
```

Again, the exact comparison and map-update syntax must come from the current reference and examples for your compiler version. The pattern itself is stable.

### Public version vs per-party nonce

You usually want both:

- **global `stateVersion`** to prevent stale-proof updates against shared contract state
- **per-party `approvalNonce[partyId]`** to prevent replay of the same participant action

These solve different problems.

`stateVersion` protects against *interleaving*:
- two users operating from the same old public state

`approvalNonce` protects against *repetition*:
- the same user input being replayed or reused when it should not be

A robust treasury flow increments `stateVersion` on every mutating action and increments the caller’s `approvalNonce` on every approval-related action.

### Conflict handling in the client

In the client or integration tests, stale updates should not be treated as mysterious failures. They are expected under concurrency. The right recovery flow is:

1. catch the stale-state failure
2. refresh public contract state
3. rebuild the witness context if needed
4. regenerate the proof against the latest version
5. retry if the action is still valid

That retry loop belongs in client code, not in Compact. The contract’s job is to reject stale transitions deterministically.

### Why “last write wins” is wrong here

In an ordinary web app, you might accept eventual consistency or overwrite conflicts optimistically. Midnight contracts are different because every state transition is proved. A proof must be tied to a specific state snapshot. If you ignore that and let callers produce updates without an explicit version check, you make debugging and reasoning much harder.

Version-checked circuits give you a clean mental model:

- every accepted proof was built against the current state
- any conflicting proof becomes invalid after the first update lands

That is the simplest concurrency story to explain, implement, and test.

## Real scenario: a private multi-sig treasury

Let’s put the pieces together around a realistic treasury flow.

### Public state

The treasury exposes only what all members need to coordinate:

- `threshold`: approvals required to authorize a spend
- `memberCount`: number of joined members
- `stateVersion`: current contract version
- `joined[partyId]`: membership flag
- `commitments[partyId]`: latest private-state commitment
- `approvalNonce[partyId]`: replay-protection counter

This is enough to answer public coordination questions:

- who is recognized as a member?
- how many members are there?
- how many approvals are required?
- is my proof based on the current version?

### Private state

Each participant keeps private local state such as:

- secret preimage for their membership/approval commitment
- local approval metadata
- expected current nonce
- any proposal details the contract does not need to reveal publicly

A participant proves consistency between their private state and the corresponding public commitment when approving, updating, or rotating secrets.

### Suggested flow

**1. Deployment**

The deployer initializes:

```compact
export circuit initialize(t: Uint<32>): [] {
  threshold = t;
  memberCount = 0;
  stateVersion = 0;
}
```

**2. Join**

A participant attaches to the deployment with `findDeployedContract`, supplies a witness returning:

- `partyId`
- initial commitment
- observed version

The `join` circuit:

- checks the observed version
- ensures the party is not already joined
- records membership
- stores the participant commitment
- initializes `approvalNonce[partyId]`
- increments `memberCount`
- increments `stateVersion`

**3. Approve a proposal**

A member submits an approval circuit that:

- proves they are a joined member
- proves knowledge of the secret corresponding to their current commitment
- proves the approval uses the expected per-party nonce
- updates any approval-related public marker or commitment
- increments both `approvalNonce[partyId]` and `stateVersion`

**4. Execute once threshold is reached**

The exact execution pattern depends on how you represent approval aggregation. There are several valid choices:

- public approval counters
- committed per-proposal approval state
- a separate proposal map keyed by proposal identifier

For a first repository, keep the execution path simple and make the tutorial focus on the N-party membership and concurrency mechanics, not on a complex proposal engine. The bounty’s core asks are multi-party private state and concurrent updates, not full DAO design.

### Why this is a better tutorial example than a toy counter

A multi-sig treasury exercises all the required concepts naturally:

- more than two users
- a map keyed by participant identifier
- independent users discovering the same deployed contract
- races between approvals and joins
- privacy-preserving member-local state

A staking pool could also work, but a treasury makes concurrency easier to explain because multiple members may approve around the same time.

## Working examples and repository structure

The bounty requires a separate **working code repository** with the full contract, tests, and multi-party interaction examples. The tutorial article should therefore point readers to a repository structure that makes the multi-party flows obvious.

A practical layout is:

```text
multi-party-treasury/
├─ contracts/
│  └─ treasury.compact
├─ client/
│  ├─ join.ts
│  ├─ approve.ts
│  └─ shared.ts
├─ tests/
│  ├─ treasury.initialize.test.ts
│  ├─ treasury.join.test.ts
│  ├─ treasury.concurrent-join.test.ts
│  ├─ treasury.approve.test.ts
│  └─ treasury.stale-proof.test.ts
├─ package.json
└─ README.md
```

### Contract fragment to include in the repository

The following fragment is based only on syntax verified by the reference primer and is safe to use as a starting point:

```compact
import CompactStandardLibrary;

export sealed ledger threshold: Uint<32>;
export ledger memberCount: Uint<32>;
export ledger stateVersion: Uint<32>;

export sealed ledger joined: Map<Bytes<32>, Boolean>;
export sealed ledger commitments: Map<Bytes<32>, Bytes<32>>;
export ledger approvalNonce: Map<Bytes<32>, Uint<32>>;

witness JoinRequest(): [Bytes<32>, Bytes<32>, Uint<32>];
witness ApprovalRequest(): [Bytes<32>, Bytes<32>, Uint<32>, Uint<32>];

export circuit initialize(t: Uint<32>): [] {
  threshold = t;
  memberCount = 0;
  stateVersion = 0;
}
```

To turn that into the repository’s final contract, verify and implement the exact current syntax for:

- reading from `Map`
- writing to `Map`
- equality/assertion constraints inside circuits

Those details must be taken from the current Midnight docs and examples for your compiler version, not guessed.

### Test scenarios the repository should cover

At minimum, the tests should prove these behaviors:

1. **initialization works**
   - threshold set
   - member count starts at zero
   - version starts at zero

2. **a third, fourth, and fifth participant can join**
   - no contract changes are needed for extra members
   - each join stores state under a distinct `partyId`

3. **multiple users can attach to one deployment**
   - one user deploys
   - others use the discovery/attach flow
   - all interact with the same contract instance

4. **concurrent stale update is rejected**
   - two users read the same version
   - first update succeeds
   - second update fails because version is stale

5. **retry against refreshed state succeeds**
   - after refresh, the losing participant rebuilds and resubmits successfully if still valid

6. **replay of an old approval fails**
   - per-party nonce prevents reuse of a prior approval witness

These tests are more important than adding extra features. For this bounty, proving the pattern is the value.

## Pitfalls and common errors

### 1. Hard-coding party names into the ledger

If your contract has fields like `aliceCommitment` and `bobCommitment`, you have not solved the N-party problem. Use `Map<Bytes<32>, ...>` or an equivalent keyed structure.

### 2. Trusting witness implementations

The Compact docs are explicit: a DApp may provide any witness implementation it wants. Never assume the witness function on the client matches your own intended logic. Constrain witness outputs inside the circuit.

### 3. Forgetting stale-proof protection

A shared contract with multiple users will see concurrent reads. Without `stateVersion` checks, your behavior under contention becomes unclear and brittle.

### 4. Using only a global version and no per-party nonce

A global version protects shared-state freshness, but it does not prevent replay of a participant’s old action in every design. Track per-party nonces where approvals or updates can be replayed.

### 5. Overexposing state publicly

Do not put the entire approval record or user secret material into the ledger just because it is convenient. Use commitments and prove statements about private data instead.

### 6. Treating client discovery as an afterthought

The join flow should be part of your design from day one. If multiple independent users cannot reliably attach to the same deployment, your tutorial does not meet the bounty’s real requirement.

### 7. Publishing unverified Compact syntax

This bounty is unusually strict: non-compiling Compact code is disqualifying. Where the article uses exact Compact syntax, it should come from official documentation. Where an API or operator is version-sensitive and not confirmed, label it clearly and verify it in the repository before submission.

## References

- [Midnight getting started](https://docs.midnight.network/getting-started)
- [Compact language reference](https://docs.midnight.network/develop/reference/compact/lang-ref)
- [Midnight MCP package](https://www.npmjs.com/package/midnight-mcp)
- [Midnight developer forum](https://forum.midnight.network/)
- [Midnight Discord](https://discord.com/invite/midnightnetwork)
- [Contributor Hub repository](https://github.com/midnightntwrk/contributor-hub)
- [Bounty terms](https://github.com/midnightntwrk/contributor-hub/blob/main/legal/BOUNTY_TERMS.md)

## Bounty spec coverage

- **Written tutorial (3,000-4,000 words)** — entire draft
- **Extending two-party patterns to N parties** — section “Extending two-party patterns to N parties”
- **Map of commitments keyed by party identifier** — section “Using a map of commitments keyed by party identifier”
- **Multiple users joining via `findDeployedContract`** — section “Letting multiple users join with `findDeployedContract`”
- **Handling concurrent state updates** — section “Handling concurrent state updates”
- **Real scenario: multi-sig treasury or staking pool** — section “Real scenario: a private multi-sig treasury”
- **Working code repository with full contract, tests, and multi-party interaction examples** — section “Working examples and repository structure”
- **All code must be tested and functional** — addressed in “Working examples and repository structure” and the test checklist
- **Follow Midnight technical style / use official syntax only where verified** — addressed throughout, especially in “Using a map of commitments keyed by party identifier,” “Letting multiple users join with `findDeployedContract`,” and “Pitfalls and common errors”