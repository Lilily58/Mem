---
name: mem-skill
description: Hybrid conversational memory management using SQLite and a vector database for retrieval-augmented user context. Use when building or updating assistants that need persistent user facts, lexical plus semantic retrieval, conflict arbitration, confidence scoring, temporal validity, and profile injection across turns.
---

# Mem Skill

Implement a memory loop with hybrid retrieval.

Use SQLite as source of truth and Chroma as vector index.

## Use Minimal Stack

Use this stack by default:

1. SQLite file (`mem.db`) for facts and history.
2. SQLite FTS5 for lexical retrieval.
3. Chroma (embedded mode) for vector retrieval.
4. LLM for extraction and arbitration.

Do not require PostgreSQL or Redis in the baseline implementation.

## Define Data Model

Store atomic facts in SQLite.

Use this minimum schema:

```json
{
  "fact_id": "uuid",
  "user_id": "string",
  "key": "string",
  "value_text": "string",
  "value_json": "json|null",
  "category": "identity|hard_preference|soft_preference|task_context|health|other",
  "confidence": 0.0,
  "event_time": "ISO-8601|null",
  "valid_from": "ISO-8601",
  "valid_to": "ISO-8601|null",
  "status": "active|superseded|expired|pending_confirmation",
  "source_turn_id": "string",
  "source_excerpt": "string",
  "embedding_id": "string|null",
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

Create archive table for superseded or expired facts.

## Build SQLite Retrieval Tables

Create these tables:

1. `facts` for active and historical fact rows.
2. `fact_history` for immutable audit records.
3. `facts_fts` as FTS5 mirror on searchable text.

Mirror `facts_fts` from `facts` with triggers or explicit sync jobs.

Index these columns in `facts`:

1. `user_id`
2. `key`
3. `status`
4. `valid_to`
5. `updated_at`

## Build Vector Collection

Use one Chroma collection for facts.

Store:

1. `id = embedding_id`
2. `document = value_text`
3. `metadata = {fact_id, user_id, key, category, status, valid_from, valid_to, confidence, updated_at}`

Update vector records whenever fact text changes.

Delete or mark vectors for facts no longer retrievable.

## Run Asynchronous Memory Observer

Queue each completed turn:

```text
(user_input, assistant_response, turn_id, timestamp, user_id)
```

Run extraction, retrieval, arbitration, and persistence in background worker.

Keep reply loop independent from observer latency.

## Stage 1: Hybrid Retrieval

Extract entities and intent from current input.

Run lexical retrieval from SQLite FTS5:

1. Query by key entities and phrases.
2. Filter by `user_id`.
3. Filter `status in (active, pending_confirmation)`.
4. Return `top_k_lex` (default `20`).

Run vector retrieval from Chroma:

1. Embed current input.
2. Query collection with user and status metadata filters.
3. Return `top_k_vec` (default `20`).

Fuse results into one candidate list.

Use default weighted score:

```text
HybridScore = 0.45*NormLexical + 0.45*VecSim + 0.10*RecencyBoost
```

Use recency boost from `updated_at` or `last_mentioned_at`.

Fallback to lexical-only retrieval when vector service fails.

## Stage 2: Evidence Arbitration

Run an internal auditor prompt over retrieved candidates.

Use this prompt template:

```text
You are a memory auditor.
Old facts:
{OLD_FACTS}

New user input:
{USER_INPUT}

Tasks:
1) Detect confirmations and repetitions.
2) Detect updates for same key.
3) Detect contradictions.
4) Decide temporary versus durable change.
5) Return JSON actions only.
```

Require output:

```json
{
  "actions": [
    {
      "type": "insert|update|supersede|expire|noop|mark_pending_confirmation",
      "key": "string",
      "new_value_text": "string|null",
      "new_value_json": "object|null",
      "target_fact_id": "uuid|null",
      "valid_from": "ISO-8601|null",
      "valid_to": "ISO-8601|null",
      "confidence_delta": 0.0,
      "reason": "string"
    }
  ]
}
```

Reject non-JSON output and re-prompt.

## Stage 3: Atomic Upsert

Apply actions in one SQLite transaction.

Use these rules:

1. Insert new fact when key does not exist.
2. Update compatible refinements in place.
3. Supersede contradictions by ending old validity and inserting new fact.
4. Expire facts when explicit time invalidation is detected.
5. Keep duplicates as noop and refresh mention timestamps and confidence.
6. Mark low-confidence conflicts as pending confirmation.

Write every destructive change to `fact_history`.

Use idempotency key:

```text
(turn_id, key, action_hash)
```

## Stage 4: Sync Embeddings

After transaction commit, sync changed facts to Chroma.

Use this policy:

1. Insert or upsert vectors for active and pending facts.
2. Remove vectors for superseded and expired facts.
3. Retry failed sync jobs with backoff.

Keep sync asynchronous so writes are not blocked by embedding calls.

## Stage 5: Materialize User Profile

Build structured profile on schedule:

1. End of session.
2. Every `N` turns (default `5`).

Project only active and valid facts:

```json
{
  "identity": {},
  "hard_preferences": {},
  "soft_preferences": {},
  "current_tasks": [],
  "recent_changes": [],
  "sensitive_topics": []
}
```

Keep confidence and timestamp fields in projection output.

## Stage 6: Budget-Aware Prompt Injection

Inject only relevant profile slices for next prompt.

Use priority:

1. Hard constraints and hard preferences.
2. Current task context.
3. Soft preferences.

Drop stale and low-confidence fields first under tight token budget.

## Confidence Policy

Use default confidence rules:

1. First mention: `0.40`.
2. Repeated or explicit confirmation: increase toward `1.00`.
3. Single contradiction against high-confidence fact: pending confirmation.

Ask clarification only for high-value conflicts.

## Operational Requirements

Track these metrics:

1. Lexical retrieval hit rate.
2. Vector retrieval hit rate.
3. Hybrid top-k recall.
4. Wrong-overwrite rate.
5. Pending-confirmation resolution rate.
6. Observer latency p50 and p95.
7. Profile injection token cost.

Run release ablations:

1. Lexical-only.
2. Vector-only.
3. Hybrid.

Publish metric deltas for each mode.

## Safety Requirements

Treat identity, health, and finance-like keys as high risk.

Use these guardrails:

1. Do not auto-overwrite high-risk facts at low confidence.
2. Require explicit confirmation when confidence is below `0.90`.
3. Keep before and after audit records for high-risk changes.
4. Support delete by `user_id` and `key`.

## Completion Checklist

Complete implementation only when all items pass:

1. SQLite schema and FTS5 retrieval are functional.
2. Chroma vector index is functional.
3. Hybrid retrieval and score fusion are implemented.
4. Arbitration returns valid JSON actions.
5. Upserts are atomic and idempotent.
6. Embedding sync runs asynchronously and retries failures.
7. Profile materialization and injection are functional.
8. High-risk guardrails are enforced.
9. Hybrid versus lexical-only metrics are reported.
