---
layout: default
title: "RISC Zero zkVM: Running Existing Rust Code Inside a STARK"
slug: risc0-zkvm-rust
ecosystem: risc0
---

# RISC Zero zkVM: Running Existing Rust Code Inside a STARK

# Risc0 zkVM for Rust developers: proving existing Rust code

If you already write Rust and you’re curious about zero-knowledge systems, Risc0 is one of the most ergonomic entry points because it lets you **prove execution of ordinary Rust code** instead of rewriting everything as arithmetic constraints.

That is the key mental shift for this tutorial:

- with a **circuit DSL**, you express a computation as constraints
- with a **zkVM**, you compile a program for a specialized virtual machine and prove that the VM executed it correctly

Risc0’s zkVM is compelling when you already have useful Rust logic,parsers, cryptographic checks, business rules, state transition code,and you want to prove *“this exact code ran on this exact input and produced this output.”*

This tutorial is aimed at Rust developers, not pure ZK newcomers. We’ll cover:

1. what a zkVM means in practice  
2. the Risc0 stack  
3. a hands-on example: take real Rust logic, move it into a guest, prove it, verify it  
4. performance tradeoffs  
5. when Risc0 wins vs circuit DSLs  
6. production examples like Bonsai and the Risc Zero bridge

---

## 1) What “zkVM” means

A zkVM is a **virtual machine whose execution can be proven cryptographically**.

With Risc0, you write a program that runs inside a deterministic RISC-V-compatible guest environment. The prover executes that program and produces a **receipt**: a proof that the program identified by some method ID ran correctly on some private input and emitted some public output.

For a Rust developer, the simplest model is:

- your Rust guest program compiles to an `rv32im` target
- the zkVM executes those instructions
- a STARK-based proof attests that execution was valid
- the verifier checks the proof and the guest’s public output

That is very different from a circuit DSL like Circom, Halo2 gadgets, or arkworks-based hand-built constraints.

### Circuit DSL model

In circuit systems, you ask:

> How do I express this computation as arithmetic constraints efficiently?

You think about:
- field arithmetic
- lookup tables
- custom gates
- bit decomposition
- witness generation
- constraint counts

That can produce extremely efficient proofs for specialized tasks, but it often means **rewriting logic** into a different form.

### zkVM model

In a zkVM, you ask:

> Can I make this computation deterministic and guest-compatible?

You think about:
- no networking, no clocks, no OS randomness
- guest memory and cycle cost
- `no_std`-friendly dependencies
- clean separation between private input and public output

The huge upside is **code reuse**. If you already have Rust that performs verification, parsing, hashing, or state transition logic, you can often reuse most of it with some refactoring.

### “STARK proof of correct execution”

At a high level, Risc0 proves:

1. the guest program with a specific image ID was loaded
2. it executed according to the VM rules
3. it consumed some private input
4. it committed some public output to the journal
5. the execution trace is valid

The verifier doesn’t re-run your program. It checks the receipt.

So instead of saying:

> Trust me, my server ran this verification.

you can say:

> Here is a receipt proving that this specific program ran and produced this result.

That is the practical meaning of “prove execution.”

---

## 2) The Risc0 stack

The Risc0 stack is easiest to understand as four layers.

## a) The zkVM / r0vm execution model

People sometimes say “r0vm” informally to mean the execution engine and VM model. The important point is that the guest runs in a **deterministic RISC-V environment**.

Your guest is not a normal Linux process. It is a VM program with limited assumptions:

- deterministic execution only
- no sockets, filesystem, or wall-clock time
- guest-compatible dependencies only
- integer-heavy execution is usually friendliest

In practice, this pushes you toward extracting a **pure core crate** with deterministic logic.

## b) The prover

The prover executes the guest and produces a receipt. Locally, that’s typically done through `risc0-zkvm` from your host application. In production, you might offload proving to **Bonsai**, Risc Zero’s remote proving service.

As a Rust developer, the prover feels like:

- build an `ExecutorEnv`
- write private input into it
- call `default_prover().prove(env, GUEST_ELF)`
- get a `receipt`

## c) The verifier

A verifier checks that the receipt corresponds to a particular guest image ID and that its journal output is authentic.

In host Rust code, that usually looks like:

```rust
receipt.verify(METHOD_ID)?;
```

After verification, you can decode the journal and trust that it really was committed by the proven guest execution.

## d) `cargo risczero`

The CLI scaffolds the project structure and build pipeline. It is the fastest way to get a real project wired correctly.

Typical flow:

```bash
cargo install cargo-risczero
cargo risczero new sigcheck-demo
```

That gives you a workspace with the usual Risc0 layout:

- `host/`: normal Rust binary that builds input, invokes the prover, verifies the receipt
- `methods/`: method embedding/build crate
- `methods/guest/`: guest program compiled for the zkVM target

For real projects, you usually add one more crate:

- `core/` or `shared/`: pure Rust logic and shared types used by both host and guest

That pattern is the sweet spot for reusing existing code.

---

## 3) Hands-on: prove a non-trivial Rust function

Let’s build a simple but non-trivial example:

- host creates a secp256k1 keypair
- host signs a message
- guest:
  - computes SHA-256 of the message
  - verifies the ECDSA signature
  - commits the digest as public output
- host verifies the receipt and reads the digest from the journal

This is a good example because it looks like real application logic: hashing + signature verification.

## Step 1: scaffold the project

Create a new Risc0 project:

```bash
cargo risczero new sigcheck-demo
cd sigcheck-demo
```

Then add a shared crate for reusable logic:

```bash
cargo new sigcheck-core --lib
```

Add `sigcheck-core` to the workspace members in the top-level `Cargo.toml`.

The directory structure becomes:

```text
sigcheck-demo/
├── Cargo.toml
├── host/
├── methods/
│   ├── build.rs
│   ├── Cargo.toml
│   └── guest/
└── sigcheck-core/
```

## Step 2: put shared logic in a reusable crate

The main trick with Risc0 is **don’t put business logic directly in the guest**. Put it in a shared crate, and let the guest be a thin wrapper.

Create `sigcheck-core/src/lib.rs`:

```rust
#![cfg_attr(not(feature = "std"), no_std)]

extern crate alloc;

use alloc::vec::Vec;
use k256::ecdsa::{signature::Verifier, Signature, VerifyingKey};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct SigInput {
    pub message: Vec<u8>,
    pub signature: [u8; 64],
    pub public_key_sec1: [u8; 33],
}

#[derive(Serialize, Deserialize, Debug, Clone, PartialEq, Eq)]
pub struct SigOutput {
    pub sha256: [u8; 32],
    pub message_len: u32,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum VerifyError {
    BadPublicKey,
    BadSignatureEncoding,
    SignatureFailed,
    MessageTooLong,
}

pub fn hash_and_verify(input: &SigInput) -> Result<SigOutput, VerifyError> {
    let digest = Sha256::digest(&input.message);
    let mut sha256 = [0u8; 32];
    sha256.copy_from_slice(&digest);

    let verifying_key = VerifyingKey::from_sec1_bytes(&input.public_key_sec1)
        .map_err(|_| VerifyError::BadPublicKey)?;

    let signature = Signature::from_slice(&input.signature)
        .map_err(|_| VerifyError::BadSignatureEncoding)?;

    verifying_key
        .verify(&input.message, &signature)
        .map_err(|_| VerifyError::SignatureFailed)?;

    let message_len =
        u32::try_from(input.message.len()).map_err(|_| VerifyError::MessageTooLong)?;

    Ok(SigOutput {
        sha256,
        message_len,
    })
}
```

This is exactly the kind of code Risc0 is good at reusing:

- ordinary Rust
- reusable data structures
- standard crypto crates
- deterministic behavior

### Dependencies for `sigcheck-core`

Your `sigcheck-core/Cargo.toml` should include:

```toml
[package]
name = "sigcheck-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
sha2 = { version = "0.10", default-features = false }
k256 = { version = "0.13", default-features = false, features = ["ecdsa"] }

[features]
default = ["std"]
std = []
```

The important part is keeping it compatible with guest compilation.

---

## Step 3: make the guest a thin wrapper

Now wire that shared logic into the zkVM guest.

Edit `methods/guest/src/bin/sigcheck.rs`:

```rust
#![no_main]

use risc0_zkvm::guest::env;
use sigcheck_core::{hash_and_verify, SigInput};

risc0_zkvm::guest::entry!(main);

pub fn main() {
    let input: SigInput = env::read();
    let output = hash_and_verify(&input).expect("signature verification failed");
    env::commit(&output);
}
```

That’s the ideal Risc0 guest:

- read private input
- call your real Rust logic
- commit public output

No application logic duplication. No circuit rewrite.

### Guest dependency

Add `sigcheck-core` as a dependency of the guest crate in `methods/guest/Cargo.toml`:

```toml
[dependencies]
risc0-zkvm = { version = "1", default-features = false }
sigcheck-core = { path = "../../sigcheck-core", default-features = false }
```

Use the version line generated by your scaffold if it is more specific. The key point is: keep the guest aligned with the versions created by `cargo risczero new`.

---

## Step 4: expose embedded method constants

The `methods` crate generated by Risc0 embeds the guest ELF and image ID into Rust constants. Keep the generated build machinery and naming conventions from the scaffold.

If your guest binary is named `sigcheck`, the generated `methods` crate will expose constants in the style of:

- `SIGCHECK_ELF`
- `SIGCHECK_ID`

The standard `methods/src/lib.rs` from the scaffold is usually enough:

```rust
include!(concat!(env!("OUT_DIR"), "/methods.rs"));
```

and `methods/build.rs` should continue to call:

```rust
fn main() {
    risc0_build::embed_methods();
}
```

In other words: let `cargo risczero` own this part.

---

## Step 5: write the host

Now the host will:

1. generate a keypair
2. sign a message
3. package the input
4. run the prover
5. verify the receipt
6. decode the journal

Create `host/src/main.rs`:

```rust
use anyhow::{Context, Result};
use k256::ecdsa::{signature::Signer, Signature, SigningKey};
use methods::{SIGCHECK_ELF, SIGCHECK_ID};
use rand_core::OsRng;
use risc0_zkvm::{default_prover, ExecutorEnv};
use sigcheck_core::{SigInput, SigOutput};

fn main() -> Result<()> {
    let message = b"Risc0 lets you prove execution of existing Rust code".to_vec();

    let signing_key = SigningKey::random(&mut OsRng);
    let verifying_key = signing_key.verifying_key();

    let signature: Signature = signing_key.sign(&message);
    let signature: [u8; 64] = signature.to_bytes().into();

    let public_key_sec1: [u8; 33] = verifying_key
        .to_encoded_point(true)
        .as_bytes()
        .try_into()
        .context("compressed secp256k1 public key should be 33 bytes")?;

    let input = SigInput {
        message,
        signature,
        public_key_sec1,
    };

    let env = ExecutorEnv::builder().write(&input)?.build()?;

    let prover = default_prover();
    let prove_info = prover.prove(env, SIGCHECK_ELF)?;
    let receipt = prove_info.receipt;

    receipt.verify(SIGCHECK_ID)?;

    let output: SigOutput = receipt.journal.decode()?;

    print!("sha256 = ");
    for byte in output.sha256 {
        print!("{byte:02x}");
    }
    println!();
    println!("message_len = {}", output.message_len);

    Ok(())
}
```

### Host dependencies

In `host/Cargo.toml`, add:

```toml
[dependencies]
anyhow = "1"
methods = { path = "../methods" }
sigcheck-core = { path = "../sigcheck-core" }
risc0-zkvm = { version = "1", features = ["prove"] }
k256 = { version = "0.13", features = ["ecdsa"] }
rand_core = { version = "0.6", features = ["getrandom"] }
```

Again, keep the `risc0-zkvm` version consistent with the scaffolded project.

---

## Step 6: build and run

Build the methods and run the host:

```bash
cargo run -p host
```

If everything is wired correctly, you should see the SHA-256 digest and message length printed after the receipt is verified.

### What just happened?

The host gave the guest a **private input** containing:

- the message
- the signature
- the public key

The guest:

- recomputed the SHA-256 digest
- verified the signature
- committed the digest and length to the journal

The receipt proves that this guest program really executed and produced that output.

The verifier does **not** need to trust the host’s claims about having run the signature check. It only needs to trust the receipt verification.

---

## A useful pattern for real codebases

The example above reflects the best pattern for migrating existing Rust code into Risc0:

### 1. Extract deterministic core logic
Move business logic into a plain library crate.

### 2. Keep guest wrappers tiny
Guests should mostly do I/O:
- read input
- call core function
- commit output

### 3. Share types between host and guest
Use `serde` types for clean input/output contracts.

### 4. Make public output explicit
Only data committed via `env::commit` becomes part of the authenticated public journal.

That architecture scales very well.

---

## 4) Performance characteristics

Risc0 performance is not about “number of constraints” in the circuit-DSl sense. Instead, you’ll mostly think in terms of:

- **guest cycle count**
- **segmenting/continuations** for larger programs
- **proof size**
- **verification cost**
- whether you prove **locally** or via **Bonsai**

## Cycle counts

The first performance question is:

> How many VM cycles does my guest need?

The answer depends heavily on the workload.

Some rough intuition:

- simple control flow and serialization: cheap
- SHA-256 on small inputs: moderate
- elliptic curve signature verification: much more expensive
- parsing large structures or doing big-int-heavy crypto: potentially very expensive

In our example, the SHA-256 part is not the dominant cost; **secp256k1 signature verification** will usually dominate.

For Rust engineers, the practical point is: Risc0 makes lots of code *possible*, but not all code *cheap*.

### What affects cycle count?

- algorithm choice
- dependency internals
- data size
- allocations and copying
- bigint / curve arithmetic
- serialization format
- branching complexity

### How to optimize

Common tactics:

- precompute outside the guest when possible
- keep guest inputs compact
- avoid unnecessary parsing in the guest
- move non-essential work to the host
- choose guest-friendly crypto crates
- prove only the minimum critical logic

A good rule is: **prove the trust boundary, not the whole application server**.

## Proof size

Raw zkVM proofs are generally larger than highly specialized SNARK proofs for the same narrow task.

That’s the tradeoff for generality.

If your workload is “prove one ECDSA verify and one hash,” a hand-optimized circuit can be much smaller and cheaper. But if your workload is “prove this entire Rust state transition function with branching and parsing,” the zkVM can be far easier to implement.

## Verifier costs

Off-chain verification of a Risc0 receipt is straightforward and usually quite practical.

On-chain verification is a different story:

- verifying large STARK-style receipts directly on-chain is not ideal
- in production, receipts are often **compressed / recursively proved**
- for EVM environments, you usually want a succinct proof format suitable for cheap smart-contract verification

That is where Risc0’s production stack matters: you generally don’t ship giant uncompressed proofs to Ethereum L1 and call it a day.

## Local proving vs Bonsai

Local proving is great for development, testing, and smaller deployments. But proving is computationally heavy. For production systems, many teams use **Bonsai** to outsource proving while still keeping verification local and trust-minimized.

Think of Bonsai as the difference between:

- compiling and running all CI on your laptop
- using a specialized remote build farm

Same receipts, different operational model.

---

## 5) Where Risc0 wins vs circuit DSLs

Risc0 is not “better than circuits” in the abstract. It is better for certain problem shapes.

## Where Risc0 wins

### Existing Rust code reuse
This is the biggest advantage.

If you already have:
- signature verification code
- Merkle logic
- transaction parsing
- consensus rules
- fraud proof/state transition code
- zk-friendly but nontrivial business logic

then Risc0 lets you **reuse architecture and tests** rather than rewrite in a circuit DSL.

### Complex branching and state machines
Circuit systems dislike dynamic, branchy, irregular control flow unless carefully engineered. A VM handles these naturally.

### Faster iteration for application teams
Most Rust teams can move much faster by editing Rust than by becoming circuit engineers.

### Easier composition with normal software
You can keep normal host-side Rust, normal crates, normal tests, and narrow the special zk-specific surface area to guest compatibility and proving.

## Where circuit DSLs still win

### Constraint efficiency
If you know exactly what computation you want and it is stable,say:
- fixed hash gadget
- fixed signature scheme
- fixed rollup arithmetic
- specialized polynomial relation

then a hand-optimized circuit will often be much cheaper.

### Small proofs / cheap on-chain verification
Specialized SNARK stacks are often superior when proof size and verifier cost are the absolute bottleneck.

### Custom math-heavy systems
If your application is fundamentally about field arithmetic, it may fit a native circuit model better than a general VM.

## The practical heuristic

Use Risc0 when:
- correctness logic already exists in Rust
- development velocity matters
- control flow is complicated
- you want to prove real software execution

Use a circuit DSL when:
- the computation is narrow and stable
- you need maximal proving/verifier efficiency
- you can afford specialized engineering

That is the honest tradeoff.

---

## 6) Production examples: Bonsai and the Risc Zero bridge

## Bonsai

**Bonsai** is Risc Zero’s proving service. Instead of running heavy proving workloads on your own machines, you send the guest image and input to Bonsai, which returns receipts.

Why this matters:

- proving is the expensive part
- verification is cheap enough to be done widely
- application teams can scale proving without building bespoke prover infrastructure

Architecturally, Bonsai lets you treat proving as an external service while keeping the security model centered on receipt verification.

That is a strong production pattern: centralized proving, decentralized verification.

## The Risc Zero bridge

The Risc Zero bridge shows where zkVMs become especially compelling: **verifying complicated protocol logic that already looks like software**.

Bridges and light-client-style systems often require:
- parsing block headers
- validating consensus rules
- checking Merkle or Patricia proofs
- processing protocol-specific branching logic
- evolving code as the source chain evolves

That is exactly the kind of workload where “prove execution of code” is attractive. Instead of expressing all of that as a bespoke circuit, you can encode the verification logic as Rust, run it in the zkVM, and produce receipts that can then be recursively compressed for on-chain consumption.

This is the pattern to keep in mind for serious systems:
- use zkVMs to prove rich off-chain logic
- recursively compress the result
- present succinct verification artifacts on-chain

---

# Final thoughts

The most important thing to understand about Risc0 is that it shifts ZK engineering from:

> designing constraints

toward:

> designing deterministic, guest-compatible Rust programs

That is a huge ergonomic advantage for Rust teams.

The workflow is:

1. extract pure Rust logic into a shared crate  
2. wrap it in a tiny guest  
3. feed input from a host  
4. prove execution  
5. verify the receipt  
6. trust only journaled output

Our hash-and-signature example is small, but it captures the real value proposition:

- you reused standard Rust crates
- you kept logic in one place
- you produced a cryptographic proof of execution

That is where Risc0 shines.

If you are evaluating it for production, the right question is not “is this more efficient than a hand-tuned circuit?” It often won’t be. The right question is:

> Is the ability to prove existing Rust execution worth the overhead?

For many applications,bridges, client proofs, verifiable backends, off-chain policy engines, state transition proofs,the answer is yes.

And if you already think in Rust modules, traits, tests, and library boundaries, Risc0 feels much more like software engineering than circuit engineering. That is its superpower.
