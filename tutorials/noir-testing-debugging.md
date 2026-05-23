---
layout: default
title: "Testing and Debugging Noir Circuits: The Honest Workflow"
slug: noir-testing-debugging
ecosystem: noir
description: "Nargo test, common Noir circuit bugs, and how to instrument proofs for fast debugging."
schema: article
og_type: article
---

# Testing and Debugging Noir Circuits: The Honest Workflow

## TL;DR

This tutorial is written against the Noir documentation snapshot fetched 2026-05-20, specifically the `dev` docs that also point readers to `v1.0.0-beta.21` as the latest version in the navigation. Everything below stays within syntax and concepts that are directly verified by the provided primer, plus standard zero-knowledge engineering practice.

The honest workflow for testing and debugging Noir circuits is:

1. **Start from `nargo new` and keep circuits small.**
2. **Use `nargo check` early** to generate `Prover.toml` and catch compile issues.
3. **Treat `nargo execute` as your first real test harness**: it compiles, executes, and writes a witness.
4. **Encode expected behavior with `assert(...)`**, because constraint failure is your ground truth.
5. **Vary inputs in `Prover.toml` methodically**, especially around boundaries and expected failures.
6. **Use public inputs intentionally** so you can reason clearly about what is revealed and what is only witness data.
7. **When debugging, reduce the circuit** to the smallest reproducer rather than guessing.
8. **If your local Noir/Nargo version exposes extra testing commands or debugging flags, verify against your local installation/compiler version.**

The key mindset: in Noir, “testing” is not only about function-level unit tests. It is also about checking whether a set of constraints admits exactly the witnesses you intend, and rejects the ones you do not.

## 1. Version, scope, and what “testing” honestly means in Noir

When developers say they want to “test” a Noir circuit, they often mean several different things at once:

- Does the program **compile**?
- Does it **execute** successfully for valid inputs?
- Does it **fail** for invalid inputs?
- Does it reveal only the values intended to be **public**?
- Does the circuit structure match the intended relation?
- Does the witness generation step behave as expected?

The provided Noir primer directly verifies a compact toolchain:

- `nargo new hello_world`
- `nargo check`
- editing `Prover.toml`
- `nargo execute`

It also verifies core language facts that matter for debugging:

- inputs are **private by default**
- inputs can be marked **public** using `pub`
- assertions are written with `assert(...)`
- Noir compiles to **ACIR**
- `nargo execute` generates a **witness**
- compiled artifacts are written under `./target/`

That is enough to describe a production-useful workflow, even without assuming any undocumented testing framework commands.

Here is the canonical starting point from the primer:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

This tiny circuit already contains the main testing ideas:

- it has a **private witness input**: `x`
- it has a **public input**: `y`
- it has a **constraint**: `x != y`

A valid run means the prover supplied a witness satisfying the constraint. An invalid run means witness generation or execution fails because the constraint system cannot be satisfied under those inputs.

That is the first honest framing: **your assertions are your executable specification**.

In conventional software, a bug often means “the code returned the wrong value.” In a ZK circuit, a bug often means one of these:

- the circuit **accepts** a witness it should reject
- the circuit **rejects** a witness it should accept
- the circuit reveals a value that should have remained private
- the circuit encodes a different relation than the protocol requires

Testing Noir circuits, then, is less about broad runtime observability and more about:

- carefully chosen input cases
- clear assertions
- minimizing ambiguity between private and public data
- inspecting the compile/execute/witness loop

If your local Nargo installation offers additional testing commands, debugger integrations, or richer failure messages, **verify against your local installation/compiler version**. This tutorial does not assume those APIs.

## 2. The basic workflow: compile, execute, and iterate

The smallest reliable workflow begins with project creation.

From the primer:

```bash
nargo new hello_world
cd hello_world
nargo check
```

The primer says `nargo check` can generate a `Prover.toml` file, where input values are specified. For the sample circuit:

```toml
x = 1
y = 2
```

Then you run:

```bash
nargo execute
```

The verified behavior is:

- `nargo execute` compiles and executes by default
- it generates the witness needed by the proving backend
- the witness is written to `./target/witness-name.gz`
- compiled artifacts are written to `./target/hello_world.json`

This gives you a clean three-stage loop:

### Stage 1: Syntax and structure

Use `nargo check` as the fastest way to catch obvious problems:

- malformed Noir code
- mismatched declarations
- issues that prevent compilation

Even if your eventual goal is proof generation, you should treat `nargo check` as the first gate on every change.

### Stage 2: Behavioral testing through execution

`nargo execute` is where a circuit starts behaving like a tested relation rather than just parsed source code. With one command you learn:

- did the program compile
- did the provided inputs satisfy the assertions
- was a witness generated successfully

This is the closest verified primitive in the primer to a test runner.

### Stage 3: Artifact awareness

Two files matter immediately:

- `./target/hello_world.json`
- `./target/witness-name.gz`

The JSON artifact represents the compiled output of the Noir program. The primer identifies Noir as compiling into ACIR. That means the artifact is part of the debugging surface: it tells you there is a concrete compiled circuit, not just source code.

The witness file matters because many ZK bugs are not source-level syntax bugs. They are witness-generation or constraint-satisfaction bugs. The fact that `nargo execute` emits a witness is evidence that, for that concrete input assignment, the relation was satisfiable.

### A practical habit: keep a “known good” input set

For any nontrivial circuit, keep at least one `Prover.toml` assignment that you know should succeed. Whenever you change the circuit:

1. run `nargo check`
2. run `nargo execute` with that known-good input
3. only then start exploring new edge cases

This helps distinguish:
- “I broke the circuit”
from
- “this new input is invalid by design”

### A practical habit: test both success and failure paths

For the sample circuit:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

You should test at least these two cases:

Valid:

```toml
x = 1
y = 2
```

Invalid:

```toml
x = 1
y = 1
```

The first case should execute successfully. The second should fail because the assertion is not satisfied.

That sounds trivial, but it is exactly the discipline that scales. Every important assertion in a circuit should have at least:

- one input set that proves it can pass when intended
- one input set that proves it fails when intended

If you do only happy-path runs, you are not really testing a circuit. You are only showing that at least one witness exists.

## 3. Write circuits so they are testable: assertions, privacy, and small invariants

A circuit that is hard to test is usually a circuit whose intended relation is not stated clearly enough. In Noir, the primary verified mechanism for stating correctness conditions in the primer is `assert(...)`.

That leads to an honest recommendation: **break correctness into small, explicit invariants**.

Start again from the verified pattern:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

What makes this testable?

- The relation is easy to describe in one sentence.
- The assertion corresponds directly to that sentence.
- A failing input is easy to generate.
- There is no ambiguity about which value is public.

Now compare that to a more realistic development mistake: combining several intended checks mentally, but writing only one of them in code. In a ZK circuit, anything not constrained is effectively unconstrained. Standard ZK engineering practice treats this as one of the most dangerous bug classes.

So when you design a Noir circuit:

### Prefer explicit assertions over implied intent

If the protocol requires several properties, constrain each property directly rather than assuming one condition implies another unless that implication is obvious and deliberate.

Even with very simple syntax, you can encode separate invariants one by one. For example:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
    assert(x != 0);
}
```

This example uses only documented Noir syntax patterns already visible in the primer:

- function definition
- `Field`
- `pub Field`
- `assert(...)`

The exact availability of every operation or literal handling should be normal in Noir, but for any nuance beyond the primer, **verify against your local installation/compiler version**.

The point is conceptual: if `x` must be different from `y`, and also must be nonzero, make both requirements explicit.

### Be deliberate about public vs private

The primer is very clear:

- all data types are private by default
- values can be marked public with `pub`
- public values are known to both prover and verifier
- changing whether a value is public affects visibility, not the logical content of the value itself

This has direct debugging consequences.

If you are confused about a circuit’s expected outputs or externally visible commitments, first ask:

- Which values are public?
- Which values are only witnesses?
- Did I accidentally reveal something by marking it `pub`?
- Did I accidentally keep something private that the verifier or contract expects as public input?

A large class of integration bugs is not “the math is wrong,” but “the visibility contract is wrong.”

For example:

```rust
fn main(secret: Field, expected: pub Field) {
    assert(secret != expected);
}
```

Here the verifier learns `expected`, not `secret`. That is easy to reason about in a test because you can vary the private witness separately from the public statement.

### Keep the first `main` tiny

For debugging, the best Noir programs often begin life as almost embarrassingly small circuits. A small `main` gives you three benefits:

1. there are fewer places for unconstrained logic to hide
2. failing inputs are easier to interpret
3. reducing to a reproducer later is much cheaper

If a circuit is becoming difficult to reason about, do not immediately reach for tool-specific debugging features. First ask whether the relation can be restated as a smaller circuit with one or two assertions that isolate the issue.

That is often faster and more reliable.

## 4. Input-case strategy: how to use `Prover.toml` as a real test harness

The primer verifies that `Prover.toml` is where input values are specified. Many Noir developers treat it as a temporary file just to get a demo running. That is a missed opportunity.

Used carefully, `Prover.toml` is your first real test harness.

For the sample program:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

you can drive several meaningful cases by changing only the input file.

### Case 1: expected success

```toml
x = 1
y = 2
```

This shows that the circuit admits a valid witness for a straightforward satisfying assignment.

### Case 2: expected failure

```toml
x = 1
y = 1
```

This shows that the same circuit rejects an assignment violating the intended relation.

### Case 3: changing only the public input

```toml
x = 5
y = 7
```

This helps you reason about how the statement seen by the verifier changes while preserving the same structure.

### Case 4: changing only the private witness

```toml
x = 8
y = 7
```

This helps you distinguish “the public statement changed” from “only the prover’s witness changed.”

That distinction matters because public and private values have different protocol meaning even when they look identical in source code.

### Test categories worth maintaining

For any nontrivial circuit, keep a short table of input classes:

- **canonical valid case**
- **canonical invalid case**
- **boundary or degenerate case**
- **privacy-sensitive case**
- **regression case** for a bug you already fixed

The primer does not define a dedicated Noir test file format or parameterized test API in the provided excerpts, so the portable workflow is to manage these input sets explicitly and rerun `nargo execute`.

If your local workflow includes scripts, CI jobs, or additional Nargo subcommands, that can be effective, but **verify against your local installation/compiler version**.

### Why this matters in ZK specifically

In standard application testing, it is often enough to validate output values. In a circuit, you are also validating the existence or nonexistence of satisfying witnesses.

That means a good test plan includes both:

- **positive tests**: there exists a witness and execution succeeds
- **negative tests**: no witness should satisfy the assertions under those inputs, so execution fails

Negative tests are especially important because under-constrained circuits tend to fail silently in design, not at syntax level. If you forgot an assertion, a bad input may still execute successfully. That is the exact behavior you are trying to detect.

### Keep failure expectations explicit

For each `Prover.toml` case, write down:

- should this execute successfully?
- if yes, which assertion or relation is it demonstrating?
- if no, which assertion should reject it?

Without that discipline, “debugging” turns into trial and error.

## 5. The debugging loop: isolate, minimize, and inspect artifacts

When a Noir circuit does not behave as expected, there are only a few honest possibilities:

1. the source code does not encode the intended relation
2. the inputs in `Prover.toml` are wrong
3. the private/public split is wrong
4. a compiler- or version-specific behavior differs from your assumption

The best debugging workflow follows that order.

### Step 1: reduce to the smallest failing example

Suppose a larger circuit fails during `nargo execute`. Your first move should not be to add more complexity. It should be to remove as much as possible while preserving the failure.

For example, if you suspect the issue is in a single inequality, create a minimal circuit:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

Then reproduce the problem with just those inputs.

This technique is standard engineering practice, but it is especially effective in ZK systems because constraint systems can be difficult to reason about globally. A small reproducer lets you verify whether the issue is:

- conceptual
- input-related
- or specific to the larger circuit composition

### Step 2: separate witness bugs from visibility bugs

If execution succeeds but the protocol semantics are wrong, ask whether the issue is actually about `pub`.

The primer distinguishes very clearly between:

- a value being public to the verifier
- a function being public to other Noir programs

Do not mix these ideas.

When debugging application-level behavior, it is often useful to temporarily simplify the circuit so that one input is clearly public and one clearly private. The sample pattern does exactly that:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

Now you can reason about whether the externally known statement is the one you meant to expose.

### Step 3: confirm the compiled artifact is being refreshed

The primer states that `nargo execute` automatically compiles if the program was not already compiled or was edited, and writes compiled artifacts under `./target/hello_world.json`.

That means artifact awareness is part of an honest workflow. If behavior seems stale or confusing:

- confirm you are running in the correct project directory
- confirm the expected source file is the one being compiled
- confirm the target artifacts correspond to the current program

The primer does not document cache-clearing or advanced build options in the provided excerpts, so avoid assuming them here.

### Step 4: treat witness generation as a debugging signal

A successful witness generation means: for the given program and inputs, the relation was satisfiable.

A failed execution means: at least one assertion or constraint prevented witness generation for those inputs.

This binary signal is more informative than it seems.

If a negative case unexpectedly succeeds, the usual explanation is not “the prover cheated.” It is much more likely that the circuit did not constrain what you thought it constrained.

If a positive case unexpectedly fails, the usual explanation is one of:

- your inputs are not actually valid
- your assertion is stronger than intended
- your circuit relation differs from the protocol relation

### Step 5: use a “spec first, then circuit” debugging mindset

Before changing code, write one sentence describing the intended relation.

For the basic sample:

> The prover knows a private `x` such that `x` is different from the public `y`.

Then compare that sentence directly to the code:

```rust
fn main(x: Field, y: pub Field) {
    assert(x != y);
}
```

If those do not match exactly, you have found the bug or at least the ambiguity.

This sounds simple, but many circuit bugs come from a drift between prose protocol definitions and encoded constraints. An honest workflow keeps both visible during debugging.

## 6. A production-minded workflow for teams

Even with only the verified Noir tooling in the primer, you can run a disciplined workflow that is suitable for serious evaluation.

### Keep circuits and input cases together

A practical project layout starts from what `nargo new` gives you:

- `src/main.nr`
- `Nargo.toml`

The primer also introduces `Prover.toml`. Treat that as part of the circuit’s executable documentation, not just a temporary local file.

For each important circuit state, keep:

- a source version of the circuit
- one or more known-good input assignments
- one or more known-bad input assignments
- a short note on expected behavior

### Use assertions as review points

During code review, every `assert(...)` should answer:

- what protocol rule does this encode?
- what invalid witness does it exclude?
- what positive example proves it is not over-constraining?

That is a more useful review discipline than only asking whether the code “looks right.”

### Don’t over-read success

A single successful `nargo execute` proves only that one witness exists for one input assignment.

It does **not** prove:

- that all invalid witnesses are rejected
- that no data is unintentionally public
- that the circuit matches off-chain business logic
- that integration with your proving backend is correct end to end

The primer explicitly says that proving and verification require a proving backend such as Barretenberg. So a prudent workflow separates:

1. **circuit-level testing in Noir**
2. **backend proving/verification testing**
3. **application integration testing**

This tutorial focuses on the first stage because that is what the provided material verifies directly.

### Add version discipline

This tutorial is written against the Noir docs snapshot fetched 2026-05-20, referencing the `dev` docs and their navigation mention of `v1.0.0-beta.21`.

That matters because tooling around testing and debugging evolves quickly in ZK ecosystems. If you see examples elsewhere using commands, flags, or APIs not shown in the provided primer, do not assume they are available in your setup. **Verify against your local installation/compiler version.**

### Know when to stop debugging Noir and start debugging protocol design

Sometimes the circuit is behaving correctly, and the real bug is in the protocol assumption.

A few warning signs:

- you cannot state the intended relation in one clear sentence
- it is unclear which inputs must be public
- different teammates give different answers about what a successful proof should mean
- your “tests” are really just random executions without expected outcomes

At that point, more CLI work will not rescue the design. Rewrite the relation, then rewrite the circuit.

## Caveats

1. **This tutorial intentionally avoids undocumented Noir APIs.**  
   The provided primer does not verify a dedicated `nargo test` command, test file conventions, tracing macros, or advanced debug output. If your local toolchain supports them, verify against your local installation/compiler version.

2. **Code examples are deliberately minimal.**  
   They use only syntax that is directly supported by the primer’s examples and language discussion: `fn main(...)`, `Field`, `pub Field`, and `assert(...)`.

3. **A successful execution is not a proof-system integration test.**  
   The primer distinguishes Noir compilation/witness generation from proving and verification, which require a backend such as Barretenberg.

4. **Public/private mistakes are common and not always obvious.**  
   The circuit may be logically correct yet still violate application privacy requirements if the wrong values are marked `pub`.

5. **Negative tests are not optional.**  
   In ZK development, a circuit that only passes valid examples may still be under-constrained. You need examples that should fail.

6. **Artifact behavior can vary by version.**  
   The primer documents `./target/hello_world.json` and `./target/witness-name.gz`, but file naming, exact formats, and surrounding tooling may change. Verify against your local installation/compiler version.

## See also

- Noir Quick Start  
  https://noir-lang.org/docs/dev/getting_started/quick_start

- Noir Data Types  
  https://noir-lang.org/docs/dev/noir/concepts/data_types/

- Noir documentation root  
  https://noir-lang.org/docs/dev/

- Barretenberg getting started entry point referenced by Noir docs  
  https://barretenberg.aztec.network/

- ACIR reference entry point from Noir docs navigation  
  https://noir-lang.org/docs/dev/ (navigate to ACIR reference in the docs sidebar)

