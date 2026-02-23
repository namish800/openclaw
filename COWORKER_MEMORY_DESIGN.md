# Memory System Design
**Date:** 2026-02-23
**Status:** Draft

## Problem

The coworker platform runs isolated sessions — channel sessions per user,
execution sessions per task, heartbeat session per agent. No single session
sees everything. An execution that onboards Alice doesn't know that the
execution that onboarded Bob hit the same NDA issue last week. The heartbeat
can check the task board but has no idea what actually happened inside those
tasks.

In OpenClaw this isn't a problem: one agent, one session, one context window.
The agent naturally accumulates experience. When it posts to the moltbook or
spots a pattern, it's drawing on conversation history it already has.

The coworker needs a memory system that bridges session isolation so the agent
can learn from its own work across executions, users, and time.

## Core Insight

The heartbeat is the agent's brain. Executions are its hands. Memory is the
connective tissue.

```
Executions generate experience
  → Activity feed captures what happened (automatic)
  → Heartbeat reads feed, reflects, spots patterns
  → Heartbeat writes memories (the insights)
  → Future executions read memories, work smarter
```

The agent gets smarter over time without anyone telling it to.

---

## Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Working Memory (per session)                      │
│  = LangGraph conversation history                           │
│  Session-scoped. Compactable. Already exists.               │
└──────────────────────────┬──────────────────────────────────┘
                     pre-compaction flush
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: Agent Memory (long-term, agent-driven)            │
│  = memories table + pgvector                                │
│  Agent explicitly saves facts, preferences, observations.   │
│  Pinned memories loaded into every session's system prompt.  │
│  Unpinned memories reachable via memory_search.             │
└─────────────────────────────────────────────────────────────┘
                           ↑
                    memory_search()
                           │
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: Activity Feed (short-term, automatic)             │
│  = activity_feed table                                      │
│  System-generated event summaries. No agent involvement.    │
│  Injected into heartbeat prompt. 14-day retention.          │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Working Memory

LangGraph conversation history per session. Already exists. Each session is
its own isolated context window.

### Pre-Compaction Flush

When any long-running session nears its context limit, a silent agent turn
fires before compaction:

**System prompt:**
> "Session nearing compaction. Save any durable facts using save_memory
> before they are compressed."

The agent calls `save_memory()` for important things, then compaction
proceeds. Tracked via `memory_flush_compaction_count` on the session to ensure
one flush per compaction cycle.

**Inspired by:** OpenClaw's `memory-flush.ts` — silent turn with
`DEFAULT_MEMORY_FLUSH_PROMPT`, fires when `totalTokens` crosses
`contextWindow - reserveTokensFloor - softThresholdTokens`.

---

## Layer 2: Agent Memory

The long-term store. Agent-driven — the agent explicitly decides what is worth
remembering, just like OpenClaw's file-based approach.

### Data Model

```
memories
┌──────────────────────────────────────────────────────────────┐
│ id                 UUID PK                                   │
│ org_id             UUID FK organizations, NOT NULL            │
│ agent_id           UUID FK agents, NOT NULL                   │
│ content            TEXT NOT NULL                              │
│ category           VARCHAR(50) NOT NULL DEFAULT 'general'     │
│                      -- fact, preference, relationship,       │
│                         observation, procedural, general      │
│ related_entity     VARCHAR(255) nullable                      │
│                      -- "user:alice", "process:onboarding",   │
│                         "task:onboard-alice"                  │
│ pinned             BOOLEAN NOT NULL DEFAULT FALSE             │
│ source_session_id  UUID nullable                              │
│ embedding          vector(1536)                               │
│ created_at         TIMESTAMPTZ NOT NULL                       │
│ updated_at         TIMESTAMPTZ NOT NULL                       │
│                                                              │
│ Indexes:                                                     │
│   (agent_id, pinned) WHERE pinned = true    -- fast load     │
│   (agent_id, category)                      -- filtered search│
│   HNSW on embedding                         -- vector search  │
└──────────────────────────────────────────────────────────────┘
```

### Pinned vs Unpinned (inspired by MEMORY.md vs memory/*.md)

OpenClaw has two tiers of memory:
- `MEMORY.md` — always loaded into the system prompt. Curated, permanent
  knowledge. Small.
- `memory/YYYY-MM-DD.md` — only reachable via `memory_search`. Running
  daily notes. Bulk of memory.

The coworker mirrors this with a `pinned` boolean:

| | Pinned | Unpinned |
|---|---|---|
| **OpenClaw equivalent** | MEMORY.md | memory/*.md |
| **Loaded into prompt** | Yes, every session | No |
| **Reachable via search** | Yes | Yes |
| **Size** | Small (dozens of entries) | Unbounded |
| **Content** | Permanent facts, core preferences | Daily observations, context, details |
| **Who promotes** | Agent calls `pin_memory()` | Default state |

**Prompt injection:** On every session start, pinned memories are loaded:
```
## Agent Memory (pinned)
- Alice is VP of Ops, prefers Slack over email
- Company fiscal year starts April 1
- NDA clause 7 causes confusion — proactively explain during onboarding
```

### Agent Tools

```
save_memory(
    content: str,
    category: str = "general",      # fact, preference, relationship,
                                      # observation, procedural
    related_to: str | None = None,   # "user:alice", "process:onboarding"
    pinned: bool = False
) -> { memory_id, content, category }
```
Creates a memory entry with embedding.

```
memory_search(
    query: str,
    category: str | None = None,
    limit: int = 6
) -> [{ memory_id, content, category, related_entity, score, created_at }]
```
Hybrid search: vector similarity (70%) + keyword BM25 (30%).
Searches across all memories for the agent (pinned and unpinned).

```
update_memory(
    memory_id: str,
    content: str | None = None,
    category: str | None = None,
    related_to: str | None = None
) -> { memory_id, content, category }
```
Updates content and re-embeds.

```
pin_memory(memory_id: str) -> { memory_id, pinned: true }
```
Promotes a memory to always-loaded status.

```
forget_memory(memory_id: str) -> { memory_id, deleted: true }
```
Soft delete (or hard delete — memories have no expiry, agent manages lifecycle).

### Search Implementation

Hybrid search combining vector similarity and keyword matching:

```python
def memory_search(agent_id, query, category=None, limit=6):
    embedding = embed(query)

    # Vector candidates (4x limit for re-ranking)
    vector_hits = db.query("""
        SELECT id, content, category, related_entity, created_at,
               1 - (embedding <=> %s) AS vector_score
        FROM memories
        WHERE agent_id = %s AND deleted_at IS NULL
        ORDER BY embedding <=> %s
        LIMIT %s
    """, embedding, agent_id, embedding, limit * 4)

    # Keyword candidates via ts_vector
    keyword_hits = db.query("""
        SELECT id, ts_rank(to_tsvector(content), plainto_tsquery(%s)) AS text_score
        FROM memories
        WHERE agent_id = %s AND deleted_at IS NULL
          AND to_tsvector(content) @@ plainto_tsquery(%s)
        LIMIT %s
    """, query, agent_id, query, limit * 4)

    # Merge and re-rank
    combined = merge(vector_hits, keyword_hits,
                     vector_weight=0.7, text_weight=0.3)
    return combined[:limit]
```

**Inspired by:** OpenClaw's `memory/manager.ts` — hybrid search with
configurable `vectorWeight` (0.7) and `textWeight` (0.3), candidate
multiplier of 4x.

---

## Layer 3: Activity Feed

Automatic, system-generated event summaries. No agent involvement in writing.
This is the short-term bridge that gives the heartbeat visibility into what
happened across all sessions.

### Data Model

```
activity_feed
┌──────────────────────────────────────────────────────────────┐
│ id                    UUID PK                                │
│ org_id                UUID FK organizations, NOT NULL         │
│ agent_id              UUID FK agents, NOT NULL                │
│ event_type            VARCHAR(50) NOT NULL                    │
│                         -- execution_completed,               │
│                            execution_failed,                  │
│                            task_status_changed,               │
│                            schedule_fired,                    │
│                            channel_interaction                │
│ summary               TEXT NOT NULL                           │
│ details               JSONB nullable                          │
│ source_session_id     UUID nullable                           │
│ source_execution_id   UUID nullable                           │
│ source_task_id        UUID nullable                           │
│ created_at            TIMESTAMPTZ NOT NULL                    │
│                                                              │
│ Index: (agent_id, created_at DESC)                           │
│ Retention: 14 days, auto-pruned via pg_cron or app sweep     │
└──────────────────────────────────────────────────────────────┘
```

### What Gets Logged

| Event | Trigger | Summary Source |
|---|---|---|
| `execution_completed` | Execution handler finishes | Last agent message or structured result |
| `execution_failed` | Execution errors | Error message + task context |
| `task_status_changed` | Task status transitions | "{title}: {old} → {new}" |
| `schedule_fired` | Scheduler fires | "{name} fired (run #{count})" |
| `channel_interaction` | Triage handles substantive msg | First sentence of agent reply (no LLM call) |

The `execution_completed` summary is the most valuable entry. The execution
handler already has the agent's final output — just extract/truncate it. No
extra LLM call needed.

For `channel_interaction`, simple heuristic: if the agent used tools or the
reply exceeded N characters, log it. Skip routine "hi"/"thanks" exchanges.

### Retention

14-day rolling window. Auto-pruned. The activity feed is short-term context,
not permanent record. Important patterns should be promoted to agent memory
by the heartbeat.

---

## Memory Strategy Per Session Type

### Channel Session (triage)

| Aspect | Strategy |
|---|---|
| **Reads at start** | Pinned memories loaded into prompt. `memory_search(user_name)` for user-specific context. |
| **Writes during** | Agent-driven. Preferences, facts about the user. |
| **Pre-compaction flush** | Yes — these accumulate across weeks of messages. |
| **Activity feed** | Auto-logs substantive interactions. |
| **System prompt guidance** | "When you learn something about this user, save it to memory." |
| **Primary memory category** | relationship, preference |

### Heartbeat Session

| Aspect | Strategy |
|---|---|
| **Reads at start** | Activity feed (since last tick) injected into prompt. Pinned memories loaded. |
| **Writes during** | Cross-session observations, patterns, insights. |
| **Pre-compaction flush** | Yes — accumulates across all ticks. |
| **Activity feed** | Consumer, not producer. |
| **System prompt guidance** | "Review recent activity. Use memory_search for deeper context. Save patterns you discover." |
| **Primary memory category** | observation, procedural |
| **Actions on insight** | Deliver message to channel, create_task, schedule_task, pin_memory |

### Execution Session

| Aspect | Strategy |
|---|---|
| **Reads at start** | Pinned memories loaded. `memory_search(task_context)` for relevant facts. |
| **Writes during** | Facts discovered during work. Procedural knowledge. |
| **Pre-compaction flush** | Yes — for long-running or multi-run executions. |
| **Activity feed** | System-generated on completion (execution_completed event). |
| **System prompt guidance** | "Search memory before starting work. Save important discoveries." |
| **Primary memory category** | fact, procedural |

### Scheduled Task Session (independent)

| Aspect | Strategy |
|---|---|
| **Reads at start** | Same as execution. Prior fires visible in session history. |
| **Writes during** | Cross-fire observations ("metric trending up since Monday"). |
| **Pre-compaction flush** | Yes — session accumulates across many fires. |
| **Activity feed** | System-generated on each fire completion. |
| **Primary memory category** | fact, observation |

### Scheduled Task Session (notify/resume)

| Aspect | Strategy |
|---|---|
| **Memory** | Inherits from target session. No separate behavior. |
| **Activity feed** | Logged as schedule_fired event. |

---

## The Value Loop

```
                   ┌─────────────────────────┐
                   │     HEARTBEAT SESSION    │
                   │  (the brain)            │
                   │                          │
                   │  Reads: activity feed    │
                   │  Reads: memory_search    │
                   │  Writes: save_memory     │
                   │  Acts: deliver, create   │
                   │        task, schedule    │
                   └──────────┬───────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │ writes          │ reads            │ acts
            ↓                 ↓                  ↓
    ┌───────────────┐ ┌──────────────┐  ┌──────────────┐
    │  MEMORY DB    │ │ MEMORY DB    │  │  TASK BOARD  │
    │  (insights)   │ │ (insights)   │  │  (new work)  │
    └───────┬───────┘ └──────────────┘  └──────┬───────┘
            │                                   │
            │ memory_search()                   │ worker picks up
            ↓                                   ↓
    ┌───────────────┐                   ┌──────────────┐
    │  EXECUTION    │                   │  EXECUTION   │
    │  SESSION      │                   │  SESSION     │
    │  (the hands)  │                   │  (the hands) │
    │               │                   │              │
    │  Reads memory │                   │ Does work    │
    │  Works smarter│                   │              │
    └───────┬───────┘                   └──────┬───────┘
            │                                   │
            │ generates                         │ generates
            ↓                                   ↓
    ┌─────────────────────────────────────────────────┐
    │              ACTIVITY FEED                       │
    │  (automatic event summaries, 14-day window)     │
    └──────────────────────┬──────────────────────────┘
                           │
                           │ heartbeat reads
                           ↓
                   ┌───────────────────────┐
                   │   HEARTBEAT SESSION   │
                   │   (next tick)         │
                   └───────────────────────┘
```

### Concrete Example: Learning Loop

```
Week 1:
  Execution "Onboard Bob"
    → Bob had questions about NDA clause 7
    → Agent writes: save_memory("Bob struggled with NDA clause 7",
        category="fact", related_to="user:bob")
    → Activity feed: "Onboard Bob: completed, NDA required extra
        clarification on clause 7"

Week 2:
  Execution "Onboard Carol"
    → Carol also got stuck on clause 7
    → Agent writes: save_memory("Carol also confused by NDA clause 7",
        category="fact", related_to="user:carol")
    → Activity feed: "Onboard Carol: completed, NDA clause 7
        questions again"

  Heartbeat tick:
    → Reads activity feed: two NDA clause 7 issues in two weeks
    → memory_search("NDA clause 7") → finds Bob and Carol entries
    → save_memory("NDA clause 7 repeatedly causes confusion during
        onboarding. Proactively explain it in welcome emails.",
        category="procedural", related_to="process:onboarding")
    → pin_memory(new_id)  -- promote to always-loaded
    → Delivers to #hr channel: "I've noticed NDA clause 7 confuses
        new hires. I'll start explaining it proactively."

Week 3:
  Execution "Onboard Dave"
    → System prompt includes pinned memory:
      "NDA clause 7 repeatedly causes confusion — proactively explain"
    → Agent includes clause 7 explanation in welcome email
    → Dave doesn't get stuck
    → Activity feed: "Onboard Dave: completed smoothly"
```

The agent learned from its own experience. No human intervention needed.

---

## System Prompt Integration

### Pinned Memories

Loaded into every session's system prompt:

```
## Agent Memory
The following are facts you have learned and pinned as important.
Refer to them as needed.

- NDA clause 7 causes confusion during onboarding — explain proactively
- Alice (VP Ops) prefers Slack, timezone US/Pacific
- Company fiscal year starts April 1
- Staging environment uses Let's Encrypt certs, renewed quarterly
```

Capped at ~20 pinned entries / ~4000 tokens to avoid bloating the prompt.
If the agent pins too many, oldest unpinned automatically (with a warning
in the next session).

### Memory Recall Guidance

Added to the system prompt when memory tools are available:

```
## Memory Recall
Before answering anything about prior work, decisions, dates, people,
preferences, or processes: run memory_search first. Then use the
results to inform your response. If nothing relevant found, proceed
normally.
```

**Inspired by:** OpenClaw's `buildMemorySection()` in `system-prompt.ts`.

### Heartbeat Prompt

```
## Recent Activity (since last heartbeat)
{formatted activity feed entries}

## Instructions
Review the activity above. Look for:
- Patterns across executions (repeated issues, common blockers)
- Overdue or stalled tasks
- Opportunities to improve processes

Use memory_search for deeper context on anything interesting.
Save observations as memories. If something needs action, create
a task or deliver a message to the relevant channel.

If nothing needs attention, reply HEARTBEAT_OK.
```

---

## Pre-Compaction Flush

Same pattern as OpenClaw, adapted for DB storage.

### Trigger

When `total_tokens` crosses:
`context_window - reserve_floor - soft_threshold`

Default: fires when ~4000 tokens remain before compaction.

### Mechanism

1. Check: `memory_flush_compaction_count != compaction_count` (one flush
   per compaction cycle)
2. Run silent agent turn with:
   - System: "Session nearing compaction. Save durable facts using
     save_memory."
   - User: "Pre-compaction flush. Store important memories now. Reply
     NO_REPLY if nothing to save."
3. Agent calls `save_memory()` for durable facts
4. Update `memory_flush_compaction_count = compaction_count`
5. Compaction proceeds normally

### Which Sessions Get Flush

| Session Type | Pre-Compaction Flush |
|---|---|
| Channel (triage) | Yes — accumulates across weeks |
| Heartbeat | Yes — accumulates across all ticks |
| Execution (long-running) | Yes — multi-run executions with deep context |
| Scheduled (independent) | Yes — accumulates across fires |
| Scheduled (notify) | No — fires into parent session, flush happens there |

---

## Component Map

| Component | File | Role |
|---|---|---|
| Memory model | `memory/models.py` | `Memory` SQLAlchemy table |
| Memory repository | `memory/repository.py` | CRUD + vector search |
| Memory service | `memory/service.py` | Business logic, embedding, search |
| Memory tools | `agent/tools/memory.py` | 5 agent tools (save, search, update, pin, forget) |
| Activity feed model | `memory/models.py` | `ActivityFeed` table |
| Activity feed writer | `memory/activity.py` | System-generated event logging |
| Activity feed pruner | `memory/activity.py` | 14-day retention sweep |
| Pre-compaction flush | `worker/memory_flush.py` | Silent turn before compaction |
| Pinned memory loader | `agent/prompt.py` | Load pinned memories into system prompt |
| Embedding service | `memory/embedding.py` | pgvector embedding generation |

---

## Configuration

```python
class MemoryConfig:
    enabled: bool = True
    embedding_model: str = "text-embedding-3-small"
    embedding_dimensions: int = 1536

    # Search
    search_max_results: int = 6
    search_min_score: float = 0.35
    search_vector_weight: float = 0.7
    search_text_weight: float = 0.3

    # Pinned
    max_pinned: int = 20
    max_pinned_tokens: int = 4000

    # Activity feed
    activity_feed_enabled: bool = True
    activity_feed_retention_days: int = 14

    # Pre-compaction flush
    flush_enabled: bool = True
    flush_soft_threshold_tokens: int = 4000
    flush_reserve_floor_tokens: int = 20000
```

---

## Dependencies

- `pgvector` extension for PostgreSQL (vector similarity search)
- OpenAI embeddings API (or compatible: Gemini, local models)
- Existing PostgreSQL instance (no new infrastructure)

---

## Future Considerations (Not In Scope)

- Memory sharing across agents within the same org
- Confidence decay / staleness scoring
- Memory deduplication (detecting near-duplicate entries)
- Memory export / import for agent migration
- RAG over external documents (company knowledge base)
