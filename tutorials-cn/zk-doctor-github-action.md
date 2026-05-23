---
layout: default
title: "5 分钟把 zk-doctor 接入你的 ZK 项目 CI"
slug: zk-doctor-github-action
ecosystem: cross
---

# 5 分钟把 zk-doctor 接入你的 ZK 项目 CI

如果你用 Compact、Leo、Noir、Cairo 或 Risc0 Rust 编写零知识(zero-knowledge, ZK)代码，你一定很熟悉一个问题：**你很难一眼判断一个 ZK 仓库是否已经达到生产可用状态**。README 看起来没问题。测试都通过了。CI 也是绿色。结果六个月后，有人尝试发布时才发现工具链没有固定版本(toolchain pinned)、缺少锁文件(lockfile)、两个检测器(detector)悄悄失败了，而且安全检查清单(security checklist)从未执行过。

这篇教程会把 [`zk-doctor`](https://github.com/Battam1111/zk-pipeline-doctor) 作为 GitHub Action 接入，这样每次 push 和每次 pull request 都会自动生成健康度报告(health report)。不需要后台守护进程(background daemon)、不需要额外账户、也没有持续费用。审计(audit)由六个彼此独立的检测器组成，覆盖语言约定(language conventions)、测试、CI、文档、安全性和可复现性(reproducibility),,最终输出为一个总分，以及一个按优先级排序、附带具体命令的修复列表。

读完本文后，你的 CI 页面会看起来和其他现代仓库一样：项目健康时显示绿色对勾，发生回退(regress)时显示红色 X，并且在 PR 上附带一个 Markdown 报告，说明该修什么。

预计耗时：如果你已经启用了 GitHub Actions，**5 分钟**即可完成。

---

## 你要添加什么

一个新文件：`.github/workflows/zk-audit.yml`。仅此而已。不需要改代码、不需要在项目里增加依赖、不需要 Dockerfile，也不需要别的东西。

这个 action（[`Battam1111/zk-doctor-action`](https://github.com/Battam1111/zk-doctor-action)）是一个复合 action(composite action),,它会在 GitHub 托管的 runner 上执行三步：

1. 设置 Python 3.11。
2. 通过 `pip` 从上游仓库(upstream repo)安装 `zk-pipeline-doctor`。
3. 对已 checkout 的代码运行 `zk-doctor` 并打印报告。

它还支持以下可选能力：

- 如果总分低于你设定的阈值(threshold)，则让 CI 失败。
- 把 Markdown 报告作为评论发布到每个 pull request。

底层 CLI 采用 MIT 许可证，你既可以固定到可变的主版本标签(mutable major tag)（`@v1`），也可以固定到不可变的补丁版本标签(immutable patch tag)（`@v1.0.0`）。

## 步骤 1 ,, 添加工作流

在你的仓库中创建 `.github/workflows/zk-audit.yml`：

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

提交并推送。这样就是最小可用安装(minimum-viable install)。在你 push 后约 20 秒内，这个 action 就会出现在 **Actions** 标签页中，并把完整的 Markdown 报告打印到运行日志(run log)里。

## 步骤 2 ,, 阅读你的第一份报告

打开工作流运行记录，点击 **audit** → **Run zk-doctor**。你会看到类似下面的内容：

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
- ⚠️ runs on a single nargo version; recommend a matrix over the last 3 releases

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

每个检测器都会给出具体命令。不会出现“可以考虑改进 X”这种泛泛而谈的提示；如果它标记了某项问题，那就意味着你确实有可以立即执行的具体修复动作。

## 步骤 3 ,, 用分数阈值为 pull request 设置门禁

当你修复了明显的缺口之后，就可以提高标准。把工作流替换为：

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

任何把分数降到 0.7 以下的 PR 都会导致 CI 失败。这确实很有用,,比如某个贡献者新增了一个没有测试的新模块，或者不小心取消了某个依赖的版本固定，都会立刻看到失败，而不是等到发布当天手忙脚乱时才发现。

根据你当前仓库的状态选择阈值：

| Current score | Suggested threshold | Rationale                                                   |
|--------------:|--------------------:|-------------------------------------------------------------|
|         < 0.5 |                 0.0 | Don't gate yet; fix the bottom first, raise the bar later  |
|     0.5 – 0.7 |                 0.5 | Lock in current state; new code can't regress               |
|     0.7 – 0.85|                 0.7 | Standard for actively-developed libraries                   |
|         ≥ 0.85|                 0.8 | High-bar for protocol-critical components                   |

## 步骤 4 ,, 在 PR 上评论报告

这对评审者(reviewers)来说是最有用的开关(flag)。添加 `output:` 和 `comment-on-pr: 'true'`，再加上 `pull-requests: write` 权限：

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

这样每个 PR 都会收到一条类似下面的评论：

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

当评审者看到这条评论时，就可以在查看 diff 的同一上下文中直接按建议采取行动。报告会在每次 push 时重新生成，因此一旦贡献者修复了问题并再次 push，这条评论就会自动更新。

## 步骤 5：通过分数在不同 PR 之间进行比较

如果你把分数发布为 job output（计划在 v1.1 中支持）或 Markdown 徽章(badge)，就可以一眼对长生命周期分支(long-running branches)做合理性检查(sanity-check)。目前，审计步骤的退出码(exit code)已经足以告诉你是否发生了回退。

## 边界情况与常见坑点

**多语言 monorepo。** zk-doctor 会按目录检测主导语言(dominant language)。如果是包含 `compact/` 和 `noir/` 子项目的 monorepo，请用不同的 `path:` 输入运行两次 action：

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

**Vendor 进仓库的依赖。** 如果你把第三方 ZK 库 vendor 到自己的仓库中，请在运行 zk-doctor 之前将其排除，否则它的质量状况会拉低你的分数。可以添加 `.zkdoctorignore`（计划在 v1.2 中支持），或者只针对某个子目录运行 action。

**自定义证明系统(proving systems)。** zk-doctor 默认识别 Plonk、Halo2、Groth16、Marlin，以及主要的 STARK 证明器(provers)。如果你构建在较少见的系统之上，语言检测器会回退到 “generic”，并跳过特定于语言的检查。其余五个检测器仍然适用。

**自托管 runner。** 这个 action 是 composite action，并使用 `actions/setup-python@v5`,,凡是支持这些的 runner 都可以运行，包括自托管环境。没有 Docker 依赖。

## 超越免费 action

这个 action 和 CLI 都采用 MIT 许可证并可免费使用。如果你想要一份**完整叙述式审计(fully-narrated audit)**，包含专家解读(expert commentary)和精致的 HTML/PDF 报告,,适合展示给安全评审人员、安全赞助方或 grant 资助方,,也有付费版本：[$99 24-hour expert audit](https://battam1111.github.io/midnight-zk-cookbook/pricing.html)（[查看示例](https://battam1111.github.io/bounty-radar-data/audits/sample.html)）。

如果你需要对多个仓库进行持续监控(ongoing monitoring)，同一套引擎也驱动着 [zk-doctor-bot](https://github.com/Battam1111/zk-doctor-bot)，这是一个 GitHub App，会对每个 PR 做更深入、由模型叙述的分析(model-narrated analysis)。定价档位(pricing tiers)也记录在同一页面上。

## 为什么这很重要

ZK 代码很难调试(debug)，更难审计(audit)，而且几乎不可能在事后补齐质量。发现缺失测试、未固定版本的依赖，或者遗漏的安全评审，成本最低的时机就是**在它进入 main 之前**。一次 5 分钟的安装，只要有一次在周五的 PR 中帮你抓到回退，就已经回本了。

如果你发现某个检测器判断错误，或者某条修复命令不适用于你的项目，请[提交 issue](https://github.com/Battam1111/zk-pipeline-doctor/issues)。检测器代码很短，补丁通常也只是单文件 PR。

## 相关阅读

- [zk-doctor CLI](https://github.com/Battam1111/zk-pipeline-doctor) ,, 上游工具，也可以在本地使用
- [bounty-radar live feed](https://battam1111.github.io/bounty-radar-data/) ,, 公开的 ZK bounty radar
- [bounty-radar-mcp](https://github.com/Battam1111/bounty-radar-mcp) ,, 从 Claude/Cursor/任意 MCP 客户端查询该 feed