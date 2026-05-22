---
layout: default
title: Midnight ZK Cookbook
description: "17 ZK tutorials across Compact, Leo, Noir, Cairo, Risc0. Plus bounty radar, pre-flight audits, and open-source tooling."
schema: breadcrumb
og_type: website
---

# The ZK Cookbook

**17 production-ready tutorials** across **5 ecosystems** (Midnight Compact, Aleo Leo, Noir/Aztec, Cairo/Starknet, RISC Zero zkVM) plus cross-ecosystem deep-dives. By [Battam1111](https://github.com/Battam1111). All examples use verified syntax from each language's official reference.

[![GitHub Sponsors](https://img.shields.io/github/sponsors/Battam1111?style=for-the-badge)](https://github.com/sponsors/Battam1111) [![Bundle](https://img.shields.io/badge/Bundle-%2415-purple?style=for-the-badge)](pricing.html#bundle) [![Radar](https://img.shields.io/badge/Radar-from%20%2419%2Fmo-blue?style=for-the-badge)](pricing.html#radar) [![Audit](https://img.shields.io/badge/Pre--Flight-%2499-green?style=for-the-badge)](pricing.html#audit)

---

## What's here

- **17 tutorials** — read freely (see below). Or grab the [$15 bundle](pricing.html#bundle) for offline reading + companion code.
- **bounty-radar feed** — live ZK bounty firehose updated every 30 min. Free to view. [$19/mo](pricing.html#radar) for real-time Telegram push + ecosystem filters.
- **zk-pipeline-doctor** — open-source CLI scoring your ZK repo on 6 dimensions. Run `pip install` and audit yourself for free. [$99 pre-flight](pricing.html#audit) if you want a human-narrated report.

## Tutorials by ecosystem

(if rendered: see structured list with descriptions below)

{% assign tutorials = site.pages | where_exp:"p","p.path contains 'tutorials/'" | sort:"title" %}
{% assign ecos = "midnight,aleo,noir,risc0,cross,devtools" | split:"," %}
{% for eco in ecos %}
{% assign eco_tuts = tutorials | where:"ecosystem", eco %}
{% if eco_tuts.size > 0 %}
### {{ eco | capitalize }}
<ul>
{% for t in eco_tuts %}
<li><a href="{{ t.url | replace: '.md', '.html' }}">{{ t.title }}</a></li>
{% endfor %}
</ul>
{% endif %}
{% endfor %}

(if you don't see a list above, your Pages build hasn't refreshed yet — try [direct tutorial listing](tutorials/).)

## Companion open-source tools

| Tool | Purpose | License |
|---|---|---|
| [zk-pipeline-doctor](https://github.com/Battam1111/zk-pipeline-doctor) | 8-detector ZK project audit CLI | MIT |
| [zk-doctor-action](https://github.com/Battam1111/zk-doctor-action) | GitHub Action wrapping the CLI | MIT |
| [bounty-radar-data](https://github.com/Battam1111/bounty-radar-data) | Live ZK bounty feed | CC0 |
| [bounty-radar-mcp](https://github.com/Battam1111/bounty-radar-mcp) | MCP server for the feed | MIT |

## Disclosure

See [DISCLOSURE.md](https://github.com/Battam1111/midnight-zk-cookbook/blob/main/DISCLOSURE.md) for our AI-assistance disclosure + human-review process.
