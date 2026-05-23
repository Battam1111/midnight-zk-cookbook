---
layout: default
title: "Building a Zero-Knowledge Identity Proof in Noir"
slug: noir-identity-proof
ecosystem: noir
description: "Zero-knowledge identity primitive in Noir: prove you are in a set without revealing which member you are."
schema: article
og_type: article
---

# Building a Zero-Knowledge Identity Proof in Noir

# Build a zero-knowledge identity proof in Noir

This tutorial builds a small but real Noir circuit for **private identity proofs**:

- the prover has a private credential,
- the credential contains a **national ID** and **age**,
- the prover shows:
  1. they know the preimage of a published credential commitment, and
  2. their age is at least 18,
- without revealing the national ID, exact age, or secret salt.

We’ll also make the proof safer for real applications by adding an **application-specific nullifier** and a **fresh challenge** so the same proof cannot be replayed.

This is a good “first real circuit” after Hello World because it forces you to think about:

- what should be **private witness data**,
- what must be **public input**,
- what the circuit actually proves,
- and what must still be enforced by the verifier contract.

---

## 1) Problem statement + threat model

Let’s define the problem precisely.

A user has a private credential made of:

- `national_id`: private
- `age`: private
- `secret`: private randomness/salt known only by the user

They have previously published a **credential commitment**:

`commitment = H(national_id, age, secret)`

Later, they want to prove to an application:

- “I own the credential behind this commitment”
- “I am 18 or older”

without revealing:

- the national ID,
- the exact age,
- or the secret.

### What the verifier learns

The verifier should only learn:

- the public `commitment`,
- that the prover is an adult,
- and a `nullifier` that prevents replay or proof re-use in the same context.

### Threat model

We want to defend against:

1. **Data leakage**  
   The verifier should not learn the raw national ID or age.

2. **Credential forgery**  
   A prover should not be able to claim someone else’s commitment without knowing its private preimage.

3. **Replay**  
   A captured proof should not be reusable by an attacker later.

4. **Proof stealing / front-running**  
   If someone sees a valid proof in mempool or transport, they should not be able to submit it for themselves.

### What this demo does *not* solve by itself

This circuit proves knowledge of a private credential and an age threshold.  
It does **not** prove that a government or issuer signed that credential.

So this is a good privacy-preserving **ownership + attribute** demo, but not yet a full verifiable-credential system. In production, you’d usually add:

- an issuer signature check,
- revocation logic,
- expiration,
- and stronger field encoding rules.

Still, this is a very realistic starting point.

---

## 2) Circuit design: constraints, witnesses, public inputs

A Noir circuit is just a function whose body becomes arithmetic constraints.

### Private witnesses

These are secret inputs known by the prover:

- `national_id: Field`
- `age: u8`
- `secret: Field`

### Public inputs

These are visible to the verifier:

- `app_id: Field`  
  Distinguishes one application from another.

- `recipient: Field`  
  The account or address this proof is intended for. We’ll bind the proof to the caller to reduce proof stealing.

- `challenge: Field`  
  A fresh nonce from the application, so each proof is unique.

- `commitment: Field`  
  The published hash of the private credential.

- `nullifier: Field`  
  A public anti-replay tag derived from secret data and application context.

### Constraints

Our circuit will enforce three things.

#### Constraint 1: age check

```text
age >= 18
```

This proves adulthood without revealing the exact age.

#### Constraint 2: commitment consistency

```text
H(national_id, age, secret) == commitment
```

This proves the user knows the secret preimage of the public commitment.

#### Constraint 3: nullifier derivation

```text
H(secret, app_id, recipient, challenge) == nullifier
```

This ties the proof to:

- a specific app,
- a specific recipient/account,
- and a fresh challenge.

That means:

- a proof for app A shouldn’t work in app B,
- a proof made for Alice shouldn’t be usable by Bob,
- a proof for challenge `n` shouldn’t work again for challenge `n+1`.

### Why include `secret` in the nullifier instead of `national_id`?

Because `secret` is a strong private trapdoor. If your ID space is small or structured, hashing only the national ID could make correlation or brute-force easier. A random secret improves unlinkability and resistance to guessing.

---

## 3) Complete Noir code

Below is the full Noir program.

### `Nargo.toml`

```toml
[package]
name = "zk_identity"
type = "bin"
authors = ["you"]
compiler_version = ">=0.30.0"
```

### `src/main.nr`

```rust
use dep::std::hash::pedersen_hash;

fn main(
    national_id: Field,
    age: u8,
    secret: Field,
    app_id: pub Field,
    recipient: pub Field,
    challenge: pub Field,
    commitment: pub Field,
    nullifier: pub Field,
) {
    // 1. Age threshold
    assert(age >= 18);

    // 2. Commitment to the private credential
    let computed_commitment = pedersen_hash([
        national_id,
        age as Field,
        secret,
    ]);

    assert(computed_commitment == commitment);

    // 3. App-specific, recipient-bound, challenge-bound nullifier
    let computed_nullifier = pedersen_hash([
        secret,
        app_id,
        recipient,
        challenge,
    ]);

    assert(computed_nullifier == nullifier);
}
```

This is intentionally small, but it is a real circuit with meaningful privacy properties.

### Why this compiles cleanly

- `Field` is Noir’s native field element type.
- `u8` gives us a bounded age value.
- `pub` marks public inputs.
- `pedersen_hash` lets us hash arrays of field elements.
- `age as Field` converts the integer into a field element so it can be hashed together with the other inputs.

---

## 4) Generating a proof + verifying on-chain

Now let’s go through the proving flow.

## Step A: compile the circuit

```bash
nargo compile
```

This creates the compiled circuit artifact in `target/`.

## Step B: prepare inputs

You need:

### Private values
- `national_id`
- `age`
- `secret`

### Public values
- `app_id`
- `recipient`
- `challenge`
- `commitment`
- `nullifier`

The important part is that `commitment` and `nullifier` must be computed with the **same formulas** as the circuit:

```text
commitment = pedersen_hash([national_id, age as Field, secret])
nullifier  = pedersen_hash([secret, app_id, recipient, challenge])
```

A common workflow is:

- compute these in your wallet/client,
- write them into `Prover.toml`,
- then run witness generation.

A typical `Prover.toml` looks like this:

```toml
national_id = "123456789"
age = "21"
secret = "987654321"
app_id = "1"
recipient = "42"
challenge = "7"
commitment = "..."
nullifier = "..."
```

Use actual field values for `commitment` and `nullifier` after computing the Pedersen hashes off-chain.

## Step C: generate the witness

```bash
nargo execute witness
```

This checks that your inputs satisfy the circuit and writes the witness data.

## Step D: create a verification key

```bash
bb write_vk -b ./target/zk_identity.json -o ./target/vk
```

## Step E: create the proof

```bash
bb prove -b ./target/zk_identity.json -w ./target/witness.gz -o ./target/proof
```

## Step F: verify locally first

```bash
bb verify -k ./target/vk -p ./target/proof
```

Always do this before trying on-chain verification.

---

## 5) Verifying on-chain with a generated verifier contract

From the verification key, generate a Solidity verifier:

```bash
bb write_solidity_verifier -k ./target/vk -o ./contracts/UltraVerifier.sol
```

This outputs a contract that can verify proofs on-chain.

### Wrapper contract

In practice, you usually don’t call the generated verifier directly.  
You wrap it in an application contract that adds business logic:

- check app ID,
- bind proof to `msg.sender`,
- enforce a fresh challenge/nonce,
- mark nullifiers as used.

Here is a minimal wrapper:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IUltraVerifier {
    function verify(bytes calldata proof, bytes32[] calldata publicInputs) external view returns (bool);
}

contract IdentityGate {
    IUltraVerifier public immutable verifier;
    uint256 public immutable appId;

    mapping(address => uint256) public nonces;
    mapping(bytes32 => bool) public usedNullifiers;
    mapping(address => bool) public isVerifiedAdult;

    event AdultVerified(address indexed account, bytes32 commitment, bytes32 nullifier);

    constructor(address _verifier, uint256 _appId) {
        verifier = IUltraVerifier(_verifier);
        appId = _appId;
    }

    function currentChallenge(address account) external view returns (uint256) {
        return nonces[account];
    }

    function proveAdult(bytes calldata proof, bytes32[] calldata publicInputs) external {
        require(publicInputs.length == 5, "bad public input length");

        // Public input ordering must match the Noir circuit:
        // app_id, recipient, challenge, commitment, nullifier
        uint256 inputAppId = uint256(publicInputs[0]);
        uint256 recipient = uint256(publicInputs[1]);
        uint256 challenge = uint256(publicInputs[2]);
        bytes32 commitment = publicInputs[3];
        bytes32 nullifier = publicInputs[4];

        require(inputAppId == appId, "wrong app id");
        require(recipient == uint256(uint160(msg.sender)), "wrong recipient");
        require(challenge == nonces[msg.sender], "wrong challenge");
        require(!usedNullifiers[nullifier], "nullifier already used");

        bool ok = verifier.verify(proof, publicInputs);
        require(ok, "invalid proof");

        usedNullifiers[nullifier] = true;
        nonces[msg.sender] += 1;
        isVerifiedAdult[msg.sender] = true;

        emit AdultVerified(msg.sender, commitment, nullifier);
    }
}
```

### Why this wrapper matters

The circuit alone proves math.  
The wrapper enforces application semantics.

- `appId` prevents cross-application reuse.
- `recipient == msg.sender` prevents someone else from stealing your proof and using it for themselves.
- `challenge == nonces[msg.sender]` makes each proof one-time and fresh.
- `usedNullifiers[nullifier]` blocks replay with the same proof.

Without these checks, a mathematically valid proof can still be unsafe in practice.

---

## 6) Common pitfalls

These are the mistakes people make most often with early Noir circuits.

### Pitfall 1: over-constraint

Over-constraint means you accidentally prove more than intended.

Example: if you made `age` public, you would reveal the exact age rather than only proving `age >= 18`.

Bad:

```rust
age: pub u8
```

Good:

```rust
age: u8
assert(age >= 18);
```

Likewise, if you expose `national_id` publicly anywhere, privacy is gone.

### Pitfall 2: under-constraint / malleability

Under-constraint means the circuit leaves room for unwanted alternate witnesses.

For example, imagine you checked `age >= 18` but forgot to bind the private data to the public commitment. Then any prover could choose *any* private age over 18 and satisfy the circuit.

This would be broken:

```rust
assert(age >= 18);
// but no commitment check
```

The commitment equality is what proves ownership of the same underlying credential.

### Pitfall 3: replay attacks

If your proof only says “I am over 18” and has no app-specific nullifier or challenge, then a copied proof might work again later.

That’s why we use:

```text
nullifier = H(secret, app_id, recipient, challenge)
```

and the contract stores used nullifiers.

### Pitfall 4: proof theft / front-running

Suppose Mallory sees Alice’s proof before it is finalized. If the proof is not bound to Alice’s account, Mallory may be able to submit it first.

Binding `recipient` into the nullifier and checking it equals `msg.sender` is a simple and effective defense.

### Pitfall 5: weak credential encoding

In this tutorial, `national_id` is a single `Field`. Real IDs often have strings, formatting, country codes, and checksums. If you compress data carelessly off-chain, different textual representations can map to the same semantic identity.

In production:

- canonicalize inputs,
- document encoding,
- and hash the canonical form only.

### Pitfall 6: small-domain guessing

If you publish a commitment over low-entropy values only, someone may brute-force them.  
For example, `H(age, birth_year)` is easy to guess.

That is why the private `secret` is so important. It turns the commitment into a salted commitment instead of a predictable one.

### Pitfall 7: assuming this proves issuer authenticity

This circuit proves:

- knowledge of secret data,
- age threshold,
- consistency with a public commitment.

It does **not** prove a government or trusted issuer vouched for that data. If you need that, add an issuer signature verification step or a Merkle inclusion proof against an issuer registry.

---

## 7) Testing with `nargo` + edge cases

A good Noir circuit should have tests before you generate proofs.

Add tests directly in `src/main.nr`.

```rust
use dep::std::hash::pedersen_hash;

fn main(
    national_id: Field,
    age: u8,
    secret: Field,
    app_id: pub Field,
    recipient: pub Field,
    challenge: pub Field,
    commitment: pub Field,
    nullifier: pub Field,
) {
    assert(age >= 18);

    let computed_commitment = pedersen_hash([
        national_id,
        age as Field,
        secret,
    ]);
    assert(computed_commitment == commitment);

    let computed_nullifier = pedersen_hash([
        secret,
        app_id,
        recipient,
        challenge,
    ]);
    assert(computed_nullifier == nullifier);
}

#[test]
fn accepts_valid_adult() {
    let national_id: Field = 123456789;
    let age: u8 = 21;
    let secret: Field = 987654321;
    let app_id: Field = 1;
    let recipient: Field = 42;
    let challenge: Field = 7;

    let commitment = pedersen_hash([
        national_id,
        age as Field,
        secret,
    ]);

    let nullifier = pedersen_hash([
        secret,
        app_id,
        recipient,
        challenge,
    ]);

    main(
        national_id,
        age,
        secret,
        app_id,
        recipient,
        challenge,
        commitment,
        nullifier,
    );
}

#[test]
fn accepts_exactly_18() {
    let national_id: Field = 1111;
    let age: u8 = 18;
    let secret: Field = 2222;
    let app_id: Field = 1;
    let recipient: Field = 99;
    let challenge: Field = 3;

    let commitment = pedersen_hash([
        national_id,
        age as Field,
        secret,
    ]);

    let nullifier = pedersen_hash([
        secret,
        app_id,
        recipient,
        challenge,
    ]);

    main(
        national_id,
        age,
        secret,
        app_id,
        recipient,
        challenge,
        commitment,
        nullifier,
    );
}

#[test(should_fail)]
fn rejects_underage() {
    let national_id: Field = 1234;
    let age: u8 = 17;
    let secret: Field = 5555;
    let app_id: Field = 1;
    let recipient: Field = 42;
    let challenge: Field = 7;

    let commitment = pedersen_hash([
        national_id,
        age as Field,
        secret,
    ]);

    let nullifier = pedersen_hash([
        secret,
        app_id,
        recipient,
        challenge,
    ]);

    main(
        national_id,
        age,
        secret,
        app_id,
        recipient,
        challenge,
        commitment,
        nullifier,
    );
}

#[test(should_fail)]
fn rejects_wrong_commitment() {
    let national_id: Field = 123456789;
    let age: u8 = 21;
    let secret: Field = 987654321;
    let app_id: Field = 1;
    let recipient: Field = 42;
    let challenge: Field = 7;

    let bad_commitment: Field = 999999;
    let nullifier = pedersen_hash([
        secret,
        app_id,
        recipient,
        challenge,
    ]);

    main(
        national_id,
        age,
        secret,
        app_id,
        recipient,
        challenge,
        bad_commitment,
        nullifier,
    );
}

#[test(should_fail)]
fn rejects_wrong_nullifier() {
    let national_id: Field = 123456789;
    let age: u8 = 21;
    let secret: Field = 987654321;
    let app_id: Field = 1;
    let recipient: Field = 42;
    let challenge: Field = 7;

    let commitment = pedersen_hash([
        national_id,
        age as Field,
        secret,
    ]);

    let bad_nullifier: Field = 123;

    main(
        national_id,
        age,
        secret,
        app_id,
        recipient,
        challenge,
        commitment,
        bad_nullifier,
    );
}
```

Run them with:

```bash
nargo test
```

### What these tests cover

- valid adult proof
- boundary case at exactly 18
- underage rejection
- wrong commitment rejection
- wrong nullifier rejection

### Additional edge cases worth testing

1. **Different recipient, same credential**  
   Should produce a different nullifier.

2. **Different challenge, same credential**  
   Should produce a different nullifier.

3. **Same credential reused in same app**  
   The circuit may still verify, but the contract should reject if the nullifier was already used.

4. **Age near max `u8`**  
   Example: 255. This is mostly a type-safety test.

5. **Zero secret**  
   Usually still valid mathematically, but you may want to reject zero off-chain as a policy.

---

## Verification checklist

Before you ship, check all of these:

- [ ] `national_id`, `age`, and `secret` are private witnesses
- [ ] `age` is constrained with `assert(age >= 18)`
- [ ] `commitment` is checked against `H(national_id, age, secret)`
- [ ] `nullifier` is checked against `H(secret, app_id, recipient, challenge)`
- [ ] `app_id` is fixed or validated by the verifier contract
- [ ] `recipient` is bound to `msg.sender`
- [ ] `challenge` is fresh and verifier-controlled
- [ ] used nullifiers are stored on-chain to block replay
- [ ] tests cover valid, boundary, and invalid cases
- [ ] you understand this proves private knowledge + age threshold, not issuer authenticity

<!-- cta-block:start -->

---

## Where to go next

Thanks for reading this far. If "Noir identity-proof circuits" connected with where you are, three concrete next steps:

### Learn more in Noir

The full [Midnight ZK Cookbook index](https://battam1111.github.io/midnight-zk-cookbook/) has 17 tutorials across Midnight, Aleo, Aztec, Noir, and risc0 plus 4 Chinese translations. Adjacent tutorials are listed by ecosystem on that page.

### Find paid work in Noir

Bounty Radar tracks open ZK bounties across Algora, GitHub labels, Drips Wave, Code4rena, and Bountycaster. Browse the [Noir sub-feed](https://battam1111.github.io/bounty-radar-data/widget.html?ecosystem=noir); JSON at [`/noir.json`](https://battam1111.github.io/bounty-radar-data/noir.json). The free tier is poll-based; the [$19/mo Hobbyist tier](https://polar.sh/checkout/polar_c_BbZbN6eJnZ7rwsUfT1pMsj4lTftwnfMoGdWBo0KozKU) pushes one filter to your Telegram in real time.

### Audit your own ZK pipeline

[zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor) is the free MIT-licensed CLI that scores any ZK project on tests, CI, docs, security, reproducibility, and language toolchain (supports Compact, Leo, Noir, Cairo, and 7 Rust zkVMs). Drop it into a GitHub Action with [zk-doctor-action](https://github.com/Battam1111/zk-doctor-action) for diff-aware PR comments. The [$15/mo Pro tier](https://github.com/Battam1111/zk-pipeline-doctor#pro-tier) adds four cross-ecosystem deep detectors (circuit complexity, proving-system pitfalls, verifier soundness, multi-file consistency).

---

*Drafted with AI assistance and reviewed by the author before publishing. See [DISCLOSURE](https://battam1111.github.io/midnight-zk-cookbook/DISCLOSURE.html) for the full process.*

<!-- cta-block:end -->
