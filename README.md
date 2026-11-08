# 🧠 Mnemochron

**Cost-efficient, time-aware knowledge-graph memory for conversational AI agents.**

[![FastAPI](https://img.shields.io/badge/Backend-FastAPI-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com)
[![Next.js](https://img.shields.io/badge/Frontend-Next.js_15-black?style=flat-square&logo=next.js)](https://nextjs.org)
[![Neo4j](https://img.shields.io/badge/Database-Neo4j_5-008CC1?style=flat-square&logo=neo4j)](https://neo4j.com)
[![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?style=flat-square&logo=python)](https://python.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript)](https://www.typescriptlang.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](#license)

> B.Tech CSE (AI & ML) major project. Mnemochron was previously named *Chronos Memory*.

[Problem](#the-problem) · [How it works](#how-it-works) · [Quickstart](#quickstart) · [Benchmark](#benchmark) · [API](#api) · [Limitations](#limitations)

---

## The problem

Chat agents that "remember" users usually store past messages as text chunks and retrieve them by similarity. That breaks down in three ways:

1. **They can't tell what's current.** Ask "Where does Alice live now?" after the memory holds both *"Alice lived in Seattle in 2020"* and *"Alice moved to London in 2023"*. A plain vector search can return both, and the agent may answer with the outdated one.
2. **Extraction is expensive.** Running an LLM on every turn to pull out facts adds token cost and latency to every message.
3. **They can't be audited.** There is no way to ask what the agent believed at a past date, or when it learned a fact versus when that fact was actually true.

## What Mnemochron does

Mnemochron stores facts as edges in a **bi-temporal knowledge graph** and handles the "what's current?" question explicitly:

- **Two-tier extraction.** A rule engine (regex, pronoun resolution, relative-time parsing) handles common statements at zero token cost. An OpenAI-compatible LLM is called only when the rules aren't confident enough.
- **Automatic contradiction handling.** When a single-valued fact changes (home city, employer, job title), the old fact is closed and linked to the new one with a `SUPERSEDES` relation.
- **Point-in-time retrieval.** Query the memory as it was at any timestamp, such as "Where did Alice live in 2021?".
- **Provenance.** Every fact records both when it was true and when the agent learned it.
- **A built-in benchmark** that compares extraction strategies on accuracy, stale-fact errors, tokens, cost and latency.

---

## How it works

```
Conversation turn (user / assistant)
            │
            ▼
┌──────────────────────────────────────────────────────────┐
│ Tier 1: rule engine + pronoun resolution                 │  0 tokens, ~ms latency
│ (regex, lexical anchors, relative-time parsing)          │
└───────────────────────────┬──────────────────────────────┘
                            │ only if confidence < τ (default 0.80)
                            ▼
┌──────────────────────────────────────────────────────────┐
│ Tier 2: LLM fallback (OpenAI-compatible, e.g. GPT-4o-mini)│  measured token accounting
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ Temporal contradiction resolution                        │
│  • detects conflicts on single-valued predicates         │
│  • sets valid_to on the old fact, is_current = false     │
│  • adds a SUPERSEDES edge with provenance                │
└───────────────────────────┬──────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────┐
│ Graph store: Neo4j (primary) + SQLite local mirror       │
└───────────────┬───────────────────────────┬──────────────┘
                ▼                           ▼
   Point-in-time retrieval          Temporal inference
   (vector + subgraph)              (reasoning chains)
```

### The bi-temporal model

Each relation edge stores:

| Field | Meaning |
|:--|:--|
| `valid_from` | When the fact became true in the real world |
| `valid_to` | When it stopped being true (`None` if still true) |
| `tx_time` | When the agent learned the fact |
| `is_current` | Whether the fact is currently active |

Point-in-time queries use `valid_from <= T AND (valid_to IS NULL OR valid_to > T)`.

### Dashboard

The Next.js frontend provides panels for ingestion, an interactive graph with a **time slider (2019 to present)**, retrieval, inference, evaluation and an audit trail.

---

## Quickstart

**Requirements:** Docker and Docker Compose. An OpenAI-compatible API key is optional. Without one, only the rule-based tier and the offline baseline run.

```bash
git clone https://github.com/lakshay3605/Major-project.git
cd Major-project

cp .env.example .env      # add OPENAI_API_KEY if you have one
docker compose up --build
```

This starts Neo4j, the FastAPI backend and the Next.js frontend. Check `docker-compose.yml` for the exact ports. The backend's interactive API docs are served at `/docs` on the backend port.

### Run the tests

```bash
cd backend
pip install -r requirements.txt
pytest
```

Current status: **45 passed, 1 skipped** (the skipped test needs a live LLM provider).

| Test file | Covers |
|:--|:--|
| `test_extraction.py` | Rule extraction, pronoun resolution, zero-token accounting |
| `test_contradiction.py` | Supersession and bi-temporal intervals |
| `test_retrieval_and_inference.py` | Time-travel queries ("in 2021" vs "now") |
| `test_temporal_research_validity.py` | Multi-step transitions, same-day changes, exact boundaries, out-of-order ingestion, provenance, ambiguous input, self-corrections |
| `test_hybrid_extraction.py` | Confidence gating, schema enforcement, retry behavior |
| `test_reproducible_benchmark.py` | Dataset schema, metric formulas, stale-fact detection |
| `test_temporal_engine.py` | Atomicity, idempotency, rollback |

An end-to-end check is also available:

```bash
python scripts/e2e_user_verification.py
```

---

## Benchmark

A reproducible pipeline compares extraction strategies on **35 labelled multi-turn scenarios** across 8 categories:

| Category | Scenarios | Example |
|:--|:-:|:--|
| Software stacks | 5 | Flask → FastAPI, AWS → GCP |
| Location changes | 5 | NYC → Berlin → Tokyo |
| Employment and roles | 5 | Promotion, company switch |
| Projects | 4 | Ownership handoff, deprecation |
| Non-conflicting additions | 5 | Python **and** Rust |
| Repeated facts | 3 | Restating a fact must not duplicate it |
| Ambiguous input | 4 | Hypotheticals and chit-chat must not overwrite facts |
| Adversarial | 4 | Self-corrections, out-of-order backfill |

Each scenario defines the turn sequence, the expected active and historical facts, the expected number of contradictions, and test queries with required and **forbidden** keywords. A forbidden keyword appearing in an answer counts as a **stale-fact error**.

### Results

| Metric | Mnemochron (hybrid, τ = 0.80) | Always-LLM baseline |
|:--|:-:|:-:|
| Overall F1 | 87.0% | 93.8% |
| Active-facts F1 | 91.1% | 94.7% |
| Historical-facts F1 | 77.8% | 91.7% |
| Query accuracy | 90.7% | 93.0% |
| Stale-fact errors | 0 | 0 |
| Total tokens | 7,647 | 27,062 |
| Token reduction | **71.7%** | n/a |
| Estimated cost | $0.00144 | $0.00525 |
| Mean ingest latency | 7.2 ms | 26.4 ms |
| Turns handled by rules alone | 71.4% | 0% |

**Reading the results:** the hybrid approach gives up about 7 points of F1 (2 points of query accuracy) in exchange for roughly 72% fewer tokens and about 3.6× lower latency. If accuracy matters more than cost, set the threshold to force LLM extraction on every turn (τ = 0.99).

### Threshold sweep

`benchmarks/experiments/threshold_sweep.py` varies the confidence threshold τ over 0.60, 0.70, 0.80, 0.90 and 0.99. Results are identical for τ between 0.60 and 0.90 (87.0% F1, 28.6% fallback). Only τ = 0.99 changes behavior, because it routes every turn to the LLM (93.8% F1, 27,062 tokens). On this dataset the rule engine's confidence scores fall into two groups, so τ = 0.80 is a sensible default, not a finely tuned optimum.

### Baselines

- **Always-LLM** uses a live OpenAI model when `OPENAI_API_KEY` is set. Without a key it uses a simulated offline baseline, and results are flagged with `tokens_is_measured = False`.
- **Mem0** and **Graphiti** run only if their packages and credentials are configured. Otherwise the runner reports `not_run` and the missing requirement instead of producing numbers.

---

## API

Benchmark endpoints:

| Method | Endpoint | Purpose |
|:--|:--|:--|
| `POST` | `/api/evaluation/benchmark?limit=35&strategies=all` | Run the benchmark and return JSON results |
| `GET` | `/api/evaluation/benchmark/latest` | Fetch the latest results |
| `GET` | `/api/evaluation/benchmark/scenarios` | Fetch the labelled dataset |

The backend also exposes routes for health, episodes, facts, graph, retrieval, inference and seed data. See the interactive docs for the full list.

---

## Repository structure

```
backend/
  app/
    api/        FastAPI routes
    core/       Configuration
    db/         Neo4j client, SQLite mirror, unified graph store
    models/     Pydantic schemas
    services/   Extraction, contradiction, retrieval, inference, evaluation
    main.py     Application entrypoint
  tests/        Pytest suite
frontend/
  src/
    app/        Next.js App Router
    components/ Header, Ingestion, Graph, Retrieval, Inference, Evaluation, Audit
    lib/        Typed API client
    types/      TypeScript types
scripts/        End-to-end verification
docker-compose.yml   Neo4j + backend + frontend
.env.example
```

---

## Limitations

- **Small, self-authored benchmark.** 35 scenarios written by the project author. The Iteration 2 improvements (72.6% → 87.0% F1) were made and measured on this same set, so they may not carry over to unseen conversations. A held-out or externally sourced test set is the most important next step.
- **Baseline cost figures may be simulated.** Without an API key, the Always-LLM numbers come from an offline simulation. Re-run with a live key before quoting the token and cost comparison.
- **Rule coverage is limited.** The rules cover locations, software stacks, employment, titles, hardware, tools, repositories and relationships. Unusual phrasing falls back to the LLM, which adds cost.
- **Third-party baselines not yet compared.** Mem0 and Graphiti are supported by the harness but have not been run.
- **Single-valued facts only.** Contradiction handling targets functional predicates such as a home city. Facts that can legitimately have several values are added, not replaced.

## Roadmap

- Expand the dataset toward ~150 scenarios, with a held-out test split
- Run Mem0 and Graphiti baselines with live credentials
- Report results with a live LLM baseline and multiple runs
- Broaden rule coverage and add a learned confidence score

## License

MIT. Developed as a B.Tech CSE (AI & ML) major project.
