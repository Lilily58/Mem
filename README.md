# Mem

Hybrid memory skill for assistants: `SQLite + FTS5 + Vector DB (Chroma)`.

This repository provides a production-oriented `SKILL.md` for building persistent, time-aware conversational memory with hybrid retrieval and conflict arbitration.

## Why This Exists

Most assistant memory implementations are either:

1. Too simple: keyword notes only.
2. Too heavy: multi-service infrastructure from day one.

`mem-skill` targets a practical middle path:

1. Keep source-of-truth in SQLite.
2. Add semantic recall with Chroma.
3. Use LLM arbitration for updates and contradictions.

## Core Capabilities

1. Persistent atomic user facts with confidence and validity windows.
2. Hybrid retrieval: lexical (`FTS5`) plus vector similarity.
3. Conflict handling: insert, update, supersede, expire, pending confirmation.
4. Atomic and idempotent upsert flow.
5. Profile materialization and budget-aware prompt injection.
6. High-risk safety guardrails for sensitive fields.

## Architecture

1. Main chat loop handles user response generation.
2. Async memory observer processes each completed turn.
3. SQLite stores `facts` and `fact_history`.
4. SQLite `FTS5` handles lexical search.
5. Chroma stores embeddings for semantic retrieval.
6. Arbitration output is persisted and synced back to vector index.

## Repository Layout

1. `SKILL.md`: executable skill specification and workflow.
2. `agents/openai.yaml`: UI metadata and default skill prompt.

## Quick Start

### Use As A Codex Skill

1. Install or copy this skill folder into your Codex skills directory.
2. Trigger with `$mem-skill` in prompts.
3. Ask Codex to scaffold implementation from the workflow in `SKILL.md`.

Example prompt:

```text
Use $mem-skill to build a memory observer with SQLite facts, FTS5 retrieval, Chroma vector search, and atomic conflict upserts.
```

### Use As An Implementation Blueprint

1. Start with SQLite schema (`facts`, `fact_history`, `facts_fts`).
2. Add embedding and Chroma collection sync.
3. Implement hybrid scoring and candidate fusion.
4. Add LLM JSON arbitration and transactional upsert.
5. Materialize profile and inject relevant slices per turn.

## Evaluation Baseline

Track at minimum:

1. Lexical retrieval hit rate.
2. Vector retrieval hit rate.
3. Hybrid top-k recall.
4. Wrong-overwrite rate.
5. Pending-confirmation resolution rate.
6. Observer latency p50 and p95.
7. Prompt injection token cost.

Run ablation with:

1. Lexical-only.
2. Vector-only.
3. Hybrid.

## Current Scope

This repo currently ships the skill specification and metadata.

It does not yet include:

1. Reference runtime implementation.
2. Dataset and benchmark scripts.
3. Demo UI or API server.

## Roadmap

1. Add minimal Python reference implementation.
2. Add reproducible benchmark harness.
3. Add sample datasets for contradiction and temporal updates.
4. Publish evaluation reports for retrieval and overwrite safety.

## License

This project is licensed under the MIT License.

See `LICENSE` for details.
