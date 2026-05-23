---
layout: default
title: "Add zk-doctor to your ZK project's CI in 5 minutes"
slug: zk-doctor-github-action
ecosystem: cross
---

# Add zk-doctor to your ZK project's CI in 5 minutes

If you write zero-knowledge code in Compact, Leo, Noir, Cairo, or Risc0 Rust, you live with a familiar problem: **you can't easily tell, at a glance, whether a ZK repo is production-ready**. The README looks fine. Tests pass. CI is green. And then six months later, someone tries to ship it and discovers the toolchain isn't pinned, the lockfile is missing, two detectors silently failed, and the security checklist was never run.

This tutorial wires up [`zk-doctor`](https://github.com/Battam1111/zk-pipeline-doctor) as a GitHub Action so every push and every pull request automatically produces a health report. No background daemon, no extra account, no recurring cost. The audit is six independent detectors over language conventions, tests, CI, docs, security, and reproducibility; emitted as a single overall score plus a prioritized fix list with concrete commands.

By the end of this post your CI page will look like every other modern repo: green checkmark when the project is healthy, red X when it regresses, with a Markdown report attached to the PR explaining what to fix.

Estimated time: **5 minutes** if you already have GitHub Actions enabled.

---

## What you're adding

A new file: `.github/workflows/zk-audit.yml`. That's it. No code changes, no dependencies in your project, no Dockerfile, nothing else.

The action ([`Battam1111/zk-doctor-action`](https://github.com/Battam1111/zk-doctor-action)) is a composite action; it runs three steps on the GitHub-hosted runner:

1. Sets up Python 3.11.
2. Installs `zk-pipeline-doctor` from the upstream repo via `pip`.
3. Runs `zk-doctor` against your checked-out code and prints the report.

Optionally, it can also:

- Fail CI if the overall score falls below a threshold you set.
- Post the Markdown report as a comment on every pull request.

The underlying CLI is MIT-licensed and you can pin to either a mutable major tag (`@v1`) or an immutable patch tag (`@v1.0.0`).

## Step 1: Add the workflow

In your repo, create `.github/workflows/zk-audit.yml`:

```yaml
name: ZK audit
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Battam1111/zk-doctor-action@v1
```

Commit and push. That's the minimum-viable install. Within ~20 seconds of your push, the action will appear in the **Actions** tab and print the full Markdown report into the run log.

## Step 2: Read your first report

Open the workflow run, click **audit** → **Run zk-doctor**. You'll see something like:

```
# zk-doctor report

Overall: 0.74

## language  (1.00)
- detected: noir (nargo.toml present)

## tests  (0.85)
- ✅ 12 tests across 3 files
- ⚠️ no `proptest` or property-based tests detected

## ci  (0.60)
- ✅ workflow `nargo-check.yml` present
- ⚠️ runs on a single nargo version — recommend a matrix over the last 3 releases

## docs  (0.50)
- ✅ README.md present
- ❌ no `CONTRIBUTING.md`
- ❌ no `examples/` directory

## security  (0.80)
- ✅ no committed secrets detected
- ⚠️ dependency `acir-rs` not pinned to a fixed version

## reproducibility  (0.83)
- ✅ Cargo.lock committed
- ⚠️ no Nix flake or fixed nargo version pin

## Top fixes
1. Pin `nargo` to a specific version. Add `nargo = "0.34.0"` to `nargo.toml` under `[package]`.
2. Add a `CONTRIBUTING.md`. Use `gh repo edit --add-topic` to discover similar repos for inspiration.
3. Run a CI matrix over `[0.32.0, 0.33.0, 0.34.0]` in `.github/workflows/nargo-check.yml`.
```

Every detector ships with concrete commands. There are no "consider improving X" lines; if it's flagged, there is something specific you can do about it.

## Step 3: Gate pull requests on a score threshold

Once you've fixed the obvious gaps, raise the bar. Replace the workflow with:

```yaml
name: ZK audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Battam1111/zk-doctor-action@v1
        with:
          threshold: '0.7'
```

Any PR that drops the score below 0.7 will fail CI. This is genuinely useful; a contributor adding a new module without tests, or accidentally unpinning a dependency, will see the failure immediately rather than during a release-day scramble.

Pick the threshold based on where your repo is today:

| Current score | Suggested threshold | Rationale                                                   |
|--------------:|--------------------:|-------------------------------------------------------------|
|         < 0.5 |                 0.0 | Don't gate yet; fix the bottom first, raise the bar later  |
|     0.5 – 0.7 |                 0.5 | Lock in current state; new code can't regress               |
|     0.7 – 0.85|                 0.7 | Standard for actively-developed libraries                   |
|         ≥ 0.85|                 0.8 | High-bar for protocol-critical components                   |

## Step 4: Comment the report on PRs

This is the most useful flag for reviewers. Add `output:` and `comment-on-pr: 'true'`, plus the `pull-requests: write` permission:

```yaml
name: ZK audit
on: [pull_request]
permissions:
  pull-requests: write
  contents: read
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Battam1111/zk-doctor-action@v1
        with:
          threshold: '0.7'
          output: 'zk-doctor-report.md'
          comment-on-pr: 'true'
```

Now every PR gets a comment that looks like:

> ## ZK Pipeline Doctor Report
>
> Overall: 0.81
>
> ### tests (0.92)
> – ✅ 14 tests
> – ⚠️ new file `circuits/range_proof.nr` has no test coverage
>
> ### Top fixes
> 1. Add at least one happy-path test for `range_proof.nr`. Suggested skeleton:
> ```rust
> #[test]
> fn test_range_proof_in_bounds() {
>     // ...
> }
> ```

When a reviewer sees the comment they can act on the suggestion in the same context as the diff. The report is regenerated on every push, so once the contributor fixes the issue and pushes again, the comment auto-updates.

## Step 5 — Compare across PRs by score

If you publish the score as a job output (planned for v1.1) or a Markdown badge, you can sanity-check long-running branches at a glance. For now, the audit step's exit code already tells you whether you're regressing.

## Edge cases & gotchas

**Multi-language monorepos.** zk-doctor detects the dominant language per directory. For a monorepo with `compact/` and `noir/` subprojects, run the action twice with different `path:` inputs:

```yaml
strategy:
  matrix:
    project: [compact, noir]
steps:
  - uses: actions/checkout@v4
  - uses: Battam1111/zk-doctor-action@v1
    with:
      path: '${{ matrix.project }}'
```

**Vendored dependencies.** If you vendor a third-party ZK library into your repo, exclude it before running zk-doctor or its quality will drag your score down. Add a `.zkdoctorignore` (planned for v1.2) or run the action against a subdirectory.

**Custom proving systems.** zk-doctor recognises Plonk, Halo2, Groth16, Marlin, and the major STARK provers by default. If you're building on a less-common system, the language detector will fall back to "generic" and skip the language-specific checks. The other five detectors still apply.

**Self-hosted runners.** The action is composite and uses `actions/setup-python@v5`: it works on any runner that supports those, including self-hosted. There are no Docker dependencies.

## Going beyond the free action

The action and the CLI are MIT-licensed and free. If you want a **fully-narrated audit** with expert commentary and a polished HTML/PDF report (useful for showing security reviewers, sponsors, or grant funders) there's a paid version: [$99 24-hour expert audit](https://battam1111.github.io/midnight-zk-cookbook/pricing.html) ([see sample](https://battam1111.github.io/bounty-radar-data/audits/sample.html)).

For ongoing monitoring across many repos, the same engine powers [zk-doctor-bot](https://github.com/Battam1111/zk-doctor-bot), a GitHub App that reviews every PR with deeper, model-narrated analysis. Pricing tiers documented on the same page.

## Why this matters

ZK code is hard to debug, harder to audit, and almost impossible to retrofit quality into. The cheapest moment to catch a missing test, an unpinned dependency, or a forgotten security review is **before it lands in main**. A 5-minute install pays for itself the first time it catches a regression on a Friday PR.

If you find a detector that's wrong or a fix command that doesn't apply to your project, [file an issue](https://github.com/Battam1111/zk-pipeline-doctor/issues). The detector code is short and patches are usually one-file PRs.

## Related reading

- [zk-doctor CLI](https://github.com/Battam1111/zk-pipeline-doctor); the upstream tool, also usable locally
- [bounty-radar live feed](https://battam1111.github.io/bounty-radar-data/); public ZK bounty radar
- [bounty-radar-mcp](https://github.com/Battam1111/bounty-radar-mcp); query the feed from Claude/Cursor/any MCP client
