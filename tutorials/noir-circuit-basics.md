---
layout: default
title: "Writing Your First Noir Circuit: Fields, Constraints, and Proofs"
slug: noir-circuit-basics
ecosystem: noir
description: "Start-from-zero Noir circuit: field arithmetic, constraints, public/private inputs, and your first proof."
schema: article
og_type: article
---

# Writing Your First Noir Circuit: Fields, Constraints, and Proofs

## TL;DR

This tutorial is written against the Noir documentation pages fetched on **2026-05-20**, specifically the **dev** docs that note the latest stable version as **v1.0.0-beta.21**. Where behavior may differ across releases, you should **verify against your local installation/compiler version**.

If you want the shortest path to a first Noir circuit:

1. Install Nargo with `noirup`.
2. Create a project with `nargo new hello_world`.
3. Put a small circuit in `src/main.nr`, for example:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

4. Run `nargo check` to generate `Prover.toml`.
5. Fill in values such as:

```toml
x = 1
y = 2
```

6. Run `nargo execute`.

At that point you have done the essential flow:

- described a computation as a **circuit**,
- introduced **Field** values,
- imposed a **constraint** with `assert`,
- generated a **witness**,
- and prepared artifacts that a proving backend can later use to produce a proof.

The important engineering point is that Noir is not “just another programming language.” It is a way to describe algebraic constraints that a proving system can check. If you keep that mental model from the beginning, the rest of the ecosystem makes much more sense.

## What Noir is actually compiling, and why `Field` comes first

The Noir primer says that every value in Noir has a type, and that **all values in Noir are fundamentally composed of Field elements**. That one sentence is the best place to start if you are evaluating whether to ship something in Noir.

In ordinary application code, you usually think in terms of machine integers, booleans, strings, arrays, and structs. In zero-knowledge circuits, the lowest-level object is generally an element of some finite field. Different proving systems may use different concrete fields, but the design pattern is standard in ZK literature: arithmetic constraints are expressed over a finite field, and the prover demonstrates knowledge of values satisfying those constraints.

Noir exposes that reality directly with the `Field` type.

A minimal Noir program from the quick start looks like this:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

There are already several ideas packed into those three lines:

- `main` is the circuit entrypoint.
- `x` and `y` are inputs.
- `Field` is the primitive algebraic type.
- `pub Field` means `y` is public.
- `assert(...)` introduces a condition that must hold.

The docs also state that **inputs in Noir are private by default**. In ZK terms, a private input is known only to the prover. A public input is known to both prover and verifier and is revealed as part of proof verification.

That distinction matters operationally:

- A **private** value contributes to the witness.
- A **public** value also participates in the statement being proven.
- Marking something public does **not** mean “this is faster” or “this is cheaper” by itself. The primer is explicit that public and private types are treated no differently except that public values are revealed in generated proofs.

For a first circuit, `Field` is the right place to start because it avoids hidden assumptions about integer range, signedness, or bit decomposition. It also matches the underlying proving model directly. Even when you later use richer types, the docs are clear that these are abstractions layered on top of field elements.

From an engineering perspective, this is the first habit worth building: when you write Noir, ask yourself not just “what does this function compute?” but also “what constraints over field elements am I asking the prover to satisfy?”

That question will keep you from writing circuits that look familiar as programs but behave unexpectedly as proofs.

## Setting up the smallest useful Noir project

The Noir quick start documents the simplest supported setup flow with **Nargo**, the CLI tool for creating, compiling, executing, and testing Noir programs.

The installation command shown in the docs is:

```bash
curl -L https://raw.githubusercontent.com/noir-lang/noirup/refs/heads/main/install | bash
noirup
```

Then initialize a new project:

```bash
nargo new hello_world
```

According to the primer, this creates at least:

- `src/main.nr`
- `Nargo.toml`

The useful way to think about these files is:

- `src/main.nr` contains your circuit logic.
- `Nargo.toml` contains project metadata and environment options.

For your first pass, replace the boilerplate circuit with the documented example:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

This is intentionally tiny, but it is not trivial. It proves a meaningful statement:

> “I know a private value `x` such that `x` is different from the public value `y`.”

That is already a zero-knowledge statement. The verifier learns `y` and learns that the prover knows some valid `x`, but the verifier does not learn `x` itself from the circuit interface.

Next, from the project directory, run:

```bash
cd hello_world
nargo check
```

The quick start says this generates a `Prover.toml` file where input values are specified. Then populate it with valid values:

```toml
x = 1
y = 2
```

And execute:

```bash
nargo execute
```

The primer states that `nargo execute` will, by default:

- compile the Noir program if needed,
- execute it,
- generate the witness,
- write the compiled artifact to `./target/hello_world.json`,
- write the witness to a file such as `./target/witness-name.gz`.

Two practical observations are worth calling out here.

First, the **witness** is not the proof. In standard ZK terminology, a witness is the collection of private values and intermediate values needed to satisfy the circuit constraints. It is the internal evidence that a prover uses when constructing a proof.

Second, the output artifact such as `hello_world.json` is not itself a proof either. It is a compiled representation of the circuit. Noir compiles to **ACIR**, and a separate proving backend consumes the compiled circuit plus witness to produce a proof.

That separation is part of Noir’s design:

- Noir: high-level circuit language
- ACIR: intermediate representation
- proving backend: concrete proof generation and verification

For a working developer, this separation is valuable because it makes your circuit code less tied to one backend. But it also means you should be disciplined about not assuming backend capabilities that the language alone does not guarantee.

## Fields, private/public inputs, and what your circuit is really saying

Let’s stay with the first example for a moment:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

To understand it clearly, break it into the mathematical statement and the privacy statement.

### The mathematical statement

The assertion is that `x != y`.

In normal software, that reads like a branch condition or runtime check. In a circuit context, it is better understood as a **required relation**. If the relation does not hold for the provided inputs, execution fails and no valid witness is produced for that input assignment.

That means your Noir program is not just a computation. It is a specification of acceptable assignments.

### The privacy statement

The type annotations tell us what is revealed:

- `x: Field` is private by default.
- `y: pub Field` is public.

The data types primer emphasizes that private values are known only to the prover, while public values are known to both prover and verifier. It also notes that public values used with smart contract verifiers should be considered known to anyone who can read the chain once the proof is verified on-chain.

That warning matters a lot in production design. New ZK developers often expose too much as public because it is convenient for integration. In practice, every public input becomes part of the externally visible statement. If a value is sensitive, keep it private unless you have a specific reason not to.

### Why `Field` is a good first choice

The docs say Noir has other primitive and compound types, but all values are fundamentally composed of fields. For a first circuit, `Field` has two advantages:

1. It maps directly to the underlying arithmetic model.
2. It avoids version-specific questions about exact behavior of other abstractions.

If you later decide to use integers or booleans, you should do so for readability or semantic clarity, not because they change the fundamental nature of the circuit. They are still compiled down to constraints over fields.

### A slightly different first circuit

Without introducing undocumented syntax, you can still explore a different statement by modifying only the assertion:

```rust
fn main(secret: Field, threshold: pub Field) {
    assert(secret != threshold);
}
```

This is the same shape, but the names clarify intent. The proof statement becomes:

> “I know a secret field element that is not equal to the public threshold.”

That may sound simple, but naming matters when you read proofs as statements. A verifier does not care that your variable was called `x`; they care what claim they are being asked to accept.

### Private by default is a design signal

The primer’s “private by default” rule is more than a convenience. It nudges developers toward the core use case of ZK systems: prove something about secret data without revealing the data itself.

As a rule of thumb for your first Noir circuits:

- make values public only if the verifier must know them,
- keep the witness surface minimal,
- and phrase the circuit in terms of the exact statement you want a verifier to trust.

That discipline will help later when your circuits are larger and the public/private boundary has real protocol consequences.

## Constraints in practice: what `assert` buys you, and what it does not

If there is one concept to get right early, it is **constraints**.

The Noir quick start uses:

```rust
assert(x != y);
```

The primer page included in your reference does not provide a full catalog of assertion behavior beyond this example, so I will stay with what is verifiable and standard.

A circuit is valuable because it constrains allowable assignments. If you accept arbitrary inputs without constraints, there is nothing meaningful to prove. The prover would always be able to produce a witness because every assignment would be acceptable.

`assert(...)` changes that. It expresses a condition that must hold. In standard circuit language, every such condition contributes to the proof relation.

### Constraints are the product, not the code

This is the biggest conceptual shift from ordinary programming.

In application code, you may think:

- input comes in,
- logic runs,
- return value comes out.

In circuit code, the real product is the set of legal assignments. Computation exists to define and connect those assignments, but the verifier ultimately checks that the prover’s witness satisfies the constraints encoded by the program.

So when you write:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

you are defining a relation roughly of the form:

> valid if and only if the prover’s chosen `x` differs from the public `y`.

The witness generation step succeeds only when inputs satisfy the relation.

### What happens when the assertion fails

The quick start shows only a successful example with:

```toml
x = 1
y = 2
```

If you instead try equal values, you should expect execution to fail because the assertion is unsatisfied. The exact error text is compiler- and version-specific, so you should **verify against your local installation/compiler version** rather than relying on a tutorial for the exact message format.

What matters conceptually is that failure to satisfy constraints prevents valid witness generation.

That property is why a later proof means anything.

### Constraints are not secrecy

A common beginner confusion is to conflate “private input” with “well-constrained input.”

Those are different dimensions:

- **private/public** determines what is revealed,
- **constraints** determine what must be true.

A private input with no meaningful constraints is still private, but the proof may not establish anything useful. Conversely, a public input can be heavily constrained and be central to the proof statement.

For a secure design, you need both:

- correct privacy boundaries,
- and strong enough constraints to encode the intended claim.

### Why tiny examples are still useful

The first Noir circuit is intentionally too small for production use, but it teaches a pattern that scales:

1. declare inputs,
2. mark some public if needed,
3. express the intended relation with assertions,
4. generate a witness only for satisfying assignments.

That exact loop remains the core of much larger systems: authentication proofs, membership proofs, recursive verification circuits, and on-chain verifiable claims all reduce to the same idea.

Your first goal is not to write a large circuit. It is to internalize that the code exists to define a proof relation.

## From execution to witness to proof: the part Noir does, and the part your backend does

The Noir quick start is explicit that after compiling and executing, you are **ready to prove**, but you still need a **proving backend**. This separation is essential to understand before you integrate Noir into a production stack.

### What Noir and Nargo do

From the primer:

- Noir is a high-level language for zero-knowledge proofs.
- It compiles your code into **ACIR**.
- It generates **witnesses** for further proof generation and verification.

In practice, from the documented flow:

```bash
nargo check
nargo execute
```

you get:

- a compiled circuit artifact such as `./target/hello_world.json`,
- a witness file such as `./target/witness-name.gz`.

These artifacts are the bridge between your source code and the proving backend.

### What the witness is

In standard ZK literature, a witness is the secret assignment demonstrating that the statement is true. In real systems, it often includes not only top-level private inputs but also intermediate values needed to satisfy internal arithmetic relations.

The important point for a Noir developer is this:

- the witness is generated from your concrete inputs,
- and a proof backend uses that witness together with the compiled circuit to construct a cryptographic proof.

So the witness is evidence for the prover, not something the verifier trusts directly.

### What the proof is

A proof is the cryptographic object that convinces a verifier that:

- there exists a witness,
- for the given public inputs,
- satisfying the compiled circuit.

The verifier does **not** need to see the private witness itself.

That is the core value proposition of ZK systems and the reason the public/private distinction in Noir matters so much.

### What the backend does

The quick start says the most common backend for Noir is **Barretenberg**, and it lists capabilities such as:

- generate proofs and verify them,
- recursive aggregation,
- generate a Solidity verifier contract,
- check and compare circuit size.

However, your provided primer does **not** include backend command syntax. Because of your instruction to avoid fabrication, I will not invent a CLI sequence here. If you want to go from witness to proof with a specific backend, you should **verify against your local installation/compiler version** and the backend’s own getting-started documentation.

That is not a minor caveat. Backend tooling often changes faster than the language tutorial examples.

### A clean mental model for the full flow

For practical engineering, keep the pipeline in this order:

1. **Write Noir source**  
   Your `main.nr` defines the proof relation.

2. **Compile to ACIR**  
   Nargo produces an intermediate representation the backend can understand.

3. **Provide concrete inputs**  
   `Prover.toml` gives values for the current run.

4. **Execute to generate witness**  
   Noir checks that the constraints are satisfiable for those values.

5. **Use a backend to prove**  
   The backend turns “compiled circuit + witness” into a cryptographic proof.

6. **Verify against public inputs**  
   The verifier checks the proof against the circuit statement and public values.

This decomposition is useful when debugging. If something fails, ask which stage failed:

- source-level logic,
- compilation,
- witness generation,
- backend proof generation,
- or verifier integration.

That will save you a lot of time compared with treating “the proof didn’t work” as one undifferentiated problem.

## A careful first workflow you can actually build on

Now that the pieces are on the table, here is a conservative first workflow for a developer evaluating Noir.

### Step 1: Create the project

```bash
nargo new hello_world
cd hello_world
```

### Step 2: Use the smallest circuit that expresses a real statement

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

This circuit is good for a first project because it includes:

- a private input,
- a public input,
- and a constraint.

That is enough to exercise the basic proving model.

### Step 3: Generate input scaffolding

```bash
nargo check
```

Per the docs, this creates `Prover.toml`.

### Step 4: Add concrete inputs

```toml
x = 1
y = 2
```

Use values that satisfy the assertion at first. When you are learning, separate “tooling works” from “my constraints are wrong.”

### Step 5: Execute

```bash
nargo execute
```

At this point you should expect:

- witness generation,
- compiled artifact generation,
- and success if your assignment satisfies the assertion.

### Step 6: Inspect the artifacts conceptually

You do not need to understand every byte of the generated files yet. But you should understand what each file is for:

- `Prover.toml`: your input assignment
- compiled artifact: the circuit description
- witness file: satisfying assignment data for the backend

### Step 7: Deliberately break the constraints

Change the inputs so the assertion is false. For example:

```toml
x = 2
y = 2
```

Then run `nargo execute` again.

You should expect failure because the circuit relation is no longer satisfied. The exact diagnostics may vary by version, so verify against your local installation.

This is an important test because it proves your circuit is not just accepting every assignment.

### Step 8: Only then move on to backend proving

Once you trust the witness-generation stage, integrate a proving backend. The primer points you to Barretenberg, but does not provide the exact commands in the material you supplied. So the correct production guidance is:

- consult the backend’s current getting-started guide,
- pin versions in your environment,
- and verify command syntax against your local installation/compiler version.

### Why this workflow is worth following

This process is intentionally slower than “copy-paste some full-stack starter repo.” But it gives you confidence in the parts that matter:

- you know what your circuit statement is,
- you know which inputs are public,
- you know a satisfying assignment can be produced,
- and you know an unsatisfying assignment fails.

For someone deciding whether to ship Noir, that confidence is worth more than a flashy end-to-end demo.

## Caveats

1. **This tutorial is version-scoped.**  
   It is written against Noir documentation fetched on **2026-05-20**, using the **dev** docs that reference **v1.0.0-beta.21** as the latest up-to-date version. If your local toolchain differs, verify behavior and command output locally.

2. **Backend commands are intentionally omitted.**  
   Your reference primer confirms that a proving backend is required and mentions Barretenberg, but it does not provide exact proof-generation CLI syntax. To avoid fabrication, the correct instruction is to **verify against your local installation/compiler version** and the backend’s current documentation.

3. **A witness is not a proof.**  
   New users often blur these together. `nargo execute` gives you witness-related artifacts and compiled circuit output; proof generation happens later through a backend.

4. **Public inputs are revealed.**  
   The Noir data types primer is explicit that public values become known to the verifier, and in on-chain settings should be considered visible to everyone with access to that chain.

5. **Do not overread small examples.**  
   The `assert(x != y)` example is useful because it is minimal, not because inequality checks are representative of all production constraints. Treat it as a learning scaffold.

6. **Avoid inventing semantics from other languages.**  
   Noir looks familiar syntactically, but it is not ordinary application code. The right model is “constraint system over field elements,” not “general-purpose program that happens to use cryptography.”

## See also

- Noir Quick Start  
  https://noir-lang.org/docs/dev/getting_started/quick_start

- Noir Data Types  
  https://noir-lang.org/docs/dev/noir/concepts/data_types/

- Noir Documentation root  
  https://noir-lang.org/docs/dev/

- ACIR reference entry point  
  https://noir-lang.org/docs/dev/

- Barretenberg getting started reference mentioned by Noir docs  
  https://noir-lang.org/docs/dev/getting_started/quick_start

- Awesome Noir reference mentioned by Noir docs  
  https://noir-lang.org/docs/dev/getting_started/quick_start

<!-- cta-block:start -->

---

## Where to go next

Thanks for reading this far. If "Noir circuit basics" connected with where you are, three concrete next steps:

### Learn more in Noir

The full [Midnight ZK Cookbook index](https://battam1111.github.io/midnight-zk-cookbook/) has 17 tutorials across Midnight, Aleo, Aztec, Noir, and risc0 plus 4 Chinese translations. Adjacent tutorials are listed by ecosystem on that page.

### Find paid work in Noir

Bounty Radar tracks open ZK bounties across Algora, GitHub labels, Drips Wave, Code4rena, and Bountycaster. Browse the [Noir sub-feed](https://battam1111.github.io/bounty-radar-data/widget.html?ecosystem=noir); JSON at [`/noir.json`](https://battam1111.github.io/bounty-radar-data/noir.json). The free tier is poll-based; the [$19/mo Hobbyist tier](https://polar.sh/checkout/polar_c_BbZbN6eJnZ7rwsUfT1pMsj4lTftwnfMoGdWBo0KozKU) pushes one filter to your Telegram in real time.

### Audit your own ZK pipeline

[zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor) is the free MIT-licensed CLI that scores any ZK project on tests, CI, docs, security, reproducibility, and language toolchain (supports Compact, Leo, Noir, Cairo, and 7 Rust zkVMs). Drop it into a GitHub Action with [zk-doctor-action](https://github.com/Battam1111/zk-doctor-action) for diff-aware PR comments. The [$15/mo Pro tier](https://github.com/Battam1111/zk-pipeline-doctor#pro-tier) adds four cross-ecosystem deep detectors (circuit complexity, proving-system pitfalls, verifier soundness, multi-file consistency).

---

*Drafted with AI assistance and reviewed by the author before publishing. See [DISCLOSURE](https://battam1111.github.io/midnight-zk-cookbook/DISCLOSURE.html) for the full process.*

<!-- cta-block:end -->
