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
  → Heartbeat writes memories to files (the insights)
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
│  Layer 2: Agent Memory (long-term, file-driven)             │
│  = memory files in sandbox + pgvector DB (synced)           │
│  Agent writes to MEMORY.md and memory/*.md using `write`.   │
│  Post-run sync persists to DB with embeddings.              │
│  Pinned (MEMORY.md) loaded into every session prompt.       │
│  Unpinned (memory/*.md) reachable via memory_search.        │
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
> "Session nearing compaction. Write any durable facts to your memory files
> before they are compressed. Use `MEMORY.md` for critical facts and
> `memory/YYYY-MM-DD.md` for daily observations."

The agent writes to memory files using the existing `write` tool, then
compaction proceeds. A post-turn sync picks up the changes and upserts them
to the DB. Tracked via `memory_flush_compaction_count` on the session to
ensure one flush per compaction cycle.

**Inspired by:** OpenClaw's `memory-flush.ts` — silent turn with
`DEFAULT_MEMORY_FLUSH_PROMPT`, fires when `totalTokens` crosses
`contextWindow - reserveTokensFloor - softThresholdTokens`.

---

## Layer 2: Agent Memory

The long-term store. Agent-driven — the agent explicitly decides what is worth
remembering by writing to files, exactly like OpenClaw.

### Writing: Files, Not Tools

The agent writes memories using the existing `write` tool. No special
`save_memory`, `update_memory`, `pin_memory`, or `forget_memory` tools.
Zero friction — the agent just writes to files in its sandbox.

**Two files, two tiers:**

| File | Tier | Loaded into prompt | Searchable |
|---|---|---|---|
| `MEMORY.md` | Pinned (curated) | Yes, every session | Yes |
| `memory/YYYY-MM-DD.md` | Unpinned (daily log) | No | Yes |

The agent writes to `MEMORY.md` for permanent, critical knowledge (like
OpenClaw's curated `MEMORY.md`). It writes to `memory/YYYY-MM-DD.md` for
daily observations, notes, and transient facts (like OpenClaw's daily logs).

To "pin" a memory: write it to `MEMORY.md`.
To "unpin" or "forget": remove it from `MEMORY.md`.
To "update": edit the file.
Promotion: move a line from `memory/*.md` into `MEMORY.md`.

All of this uses the same `write`/`edit` tools the agent already has.

### Post-Run Sync (files → DB)

After every agent turn completes, a sync step reads the memory files from
the sandbox, diffs against the DB, and upserts:

```
Post-run sync:
1. Read MEMORY.md from sandbox
2. Parse into individual entries (one per line/block)
3. Diff against existing pinned=true rows in DB
   - New lines → INSERT with pinned=true, generate embedding
   - Removed lines → SET pinned=false (soft-unpin, entry stays searchable)
   - Changed lines → UPDATE content + re-embed
4. Read memory/*.md files from sandbox
5. Diff against existing pinned=false rows for matching dates
   - New content → INSERT with pinned=false, generate embedding
   - Changed content → UPDATE + re-embed
6. Batch embed all new/changed entries in one API call
```

This keeps the DB in sync as the source of truth for search, while the
files remain the agent's writing surface. The agent never needs to know
about the DB, embeddings, or vector search internals.

### Data Model

```
memories
┌──────────────────────────────────────────────────────────────┐
│ id                 UUID PK                                   │
│ org_id             UUID FK organizations, NOT NULL            │
│ agent_id           UUID FK agents, NOT NULL                   │
│ content            TEXT NOT NULL                              │
│ file_path          VARCHAR(255) NOT NULL                      │
│                      -- "MEMORY.md" or "memory/2026-02-23.md" │
│ line_hash          VARCHAR(64) NOT NULL                       │
│                      -- SHA-256 of content, for diffing       │
│ pinned             BOOLEAN NOT NULL DEFAULT FALSE             │
│ source_session_id  UUID nullable                              │
│ embedding          vector(1536)                               │
│ created_at         TIMESTAMPTZ NOT NULL                       │
│ updated_at         TIMESTAMPTZ NOT NULL                       │
│                                                              │
│ Indexes:                                                     │
│   (agent_id, pinned) WHERE pinned = true    -- fast load     │
│   (agent_id, file_path)                     -- sync diffing  │
│   HNSW on embedding                         -- vector search  │
└──────────────────────────────────────────────────────────────┘
```

Note: No `category` or `related_entity` columns — they added friction to the
writing experience. The agent can express structure naturally in the file
content itself. Search is semantic (vector + keyword), so categories aren't
needed for retrieval.

### Pinned vs Unpinned (inspired by MEMORY.md vs memory/*.md)

OpenClaw has two tiers of memory:
- `MEMORY.md` — always loaded into the system prompt. Curated, permanent
  knowledge. Small.
- `memory/YYYY-MM-DD.md` — only reachable via `memory_search`. Running
  daily notes. Bulk of memory.

The coworker mirrors this exactly:

| | Pinned (MEMORY.md) | Unpinned (memory/*.md) |
|---|---|---|
| **OpenClaw equivalent** | MEMORY.md | memory/*.md |
| **Loaded into prompt** | Yes, every session | No |
| **Reachable via search** | Yes | Yes |
| **Size** | Small (~4000 tokens max) | Unbounded |
| **Content** | Permanent facts, core preferences | Daily observations, context, details |
| **How to promote** | Agent edits MEMORY.md | Agent moves line from daily log to MEMORY.md |

**Prompt injection:** On every session start, contents of `MEMORY.md` are
loaded verbatim:
```
## Agent Memory
The following are facts you have learned and saved as important.
Refer to them as needed.

- NDA clause 7 causes confusion during onboarding — explain proactively
- Alice (VP Ops) prefers Slack, timezone US/Pacific
- Company fiscal year starts April 1
- Staging environment uses Let's Encrypt certs, renewed quarterly
```

Capped at ~4000 tokens to avoid bloating the prompt. If `MEMORY.md` grows
too large, the agent is prompted to curate it.

### Agent Tools (reading only)

Writing uses the existing `write`/`edit` tools. Only one memory-specific
tool exists — for searching:

```
memory_search(
    query: str,
    limit: int = 6
) -> [{ content, file_path, score, created_at }]
```
Hybrid search: vector similarity (70%) + keyword BM25 (30%).
Searches across all memories for the agent (pinned and unpinned).

### Search Implementation

Hybrid search combining vector similarity and keyword matching:

```python
def memory_search(agent_id, query, limit=6):
    embedding = embed(query)

    # Vector candidates (4x limit for re-ranking)
    vector_hits = db.query("""
        SELECT id, content, file_path, created_at,
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
by the heartbeat (by writing to memory files).

---

## Memory Strategy Per Session Type

### Channel Session (triage)

| Aspect | Strategy |
|---|---|
| **Reads at start** | `MEMORY.md` loaded into prompt. `memory_search(user_name)` for user-specific context. |
| **Writes during** | Agent writes to `memory/YYYY-MM-DD.md` — preferences, facts about the user. |
| **Pre-compaction flush** | Yes — these accumulate across weeks of messages. |
| **Activity feed** | Auto-logs substantive interactions. |
| **System prompt guidance** | "When you learn something about this user, write it to your memory files." |

### Heartbeat Session

| Aspect | Strategy |
|---|---|
| **Reads at start** | Activity feed (since last tick) injected into prompt. `MEMORY.md` loaded. |
| **Writes during** | Cross-session observations, patterns, insights to `memory/YYYY-MM-DD.md`. Promotes critical findings to `MEMORY.md`. |
| **Pre-compaction flush** | Yes — accumulates across all ticks. |
| **Activity feed** | Consumer, not producer. |
| **System prompt guidance** | "Review recent activity. Use memory_search for deeper context. Write patterns to memory files. Add critical insights to MEMORY.md." |
| **Actions on insight** | Deliver message to channel, create_task, schedule_task |

### Execution Session

| Aspect | Strategy |
|---|---|
| **Reads at start** | `MEMORY.md` loaded. `memory_search(task_context)` for relevant facts. |
| **Writes during** | Facts discovered during work to `memory/YYYY-MM-DD.md`. |
| **Pre-compaction flush** | Yes — for long-running or multi-run executions. |
| **Activity feed** | System-generated on completion (execution_completed event). |
| **System prompt guidance** | "Search memory before starting work. Write important discoveries to your memory files." |

### Scheduled Task Session (independent)

| Aspect | Strategy |
|---|---|
| **Reads at start** | Same as execution. Prior fires visible in session history. |
| **Writes during** | Cross-fire observations to `memory/YYYY-MM-DD.md` ("metric trending up since Monday"). |
| **Pre-compaction flush** | Yes — session accumulates across many fires. |
| **Activity feed** | System-generated on each fire completion. |

### Scheduled Task Session (notify/resume)

| Aspect | Strategy |
|---|---|
| **Memory** | Inherits from target session. No separate behavior. |
| **Activity feed** | Logged as schedule_fired event. |

---

## The Value Loop

```
                   ┌─────────────────────────────┐
                   │     HEARTBEAT SESSION        │
                   │  (the brain)                │
                   │                              │
                   │  Reads: activity feed        │
                   │  Reads: memory_search        │
                   │  Writes: MEMORY.md,          │
                   │          memory/*.md          │
                   │  Acts: deliver, create       │
                   │        task, schedule        │
                   └──────────┬───────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │ writes files    │ reads            │ acts
            ↓                 ↓                  ↓
    ┌───────────────┐ ┌──────────────┐  ┌──────────────┐
    │  MEMORY FILES │ │ MEMORY DB    │  │  TASK BOARD  │
    │  (sandbox)    │ │ (synced)     │  │  (new work)  │
    │       ↓       │ │              │  │              │
    │  post-run     │ │              │  │              │
    │  sync → DB    │ │              │  │              │
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
    → Agent writes to memory/2026-02-17.md:
        "Bob struggled with NDA clause 7 during onboarding"
    → Activity feed: "Onboard Bob: completed, NDA required extra
        clarification on clause 7"

Week 2:
  Execution "Onboard Carol"
    → Carol also got stuck on clause 7
    → Agent writes to memory/2026-02-23.md:
        "Carol also confused by NDA clause 7 during onboarding"
    → Activity feed: "Onboard Carol: completed, NDA clause 7
        questions again"

  Heartbeat tick:
    → Reads activity feed: two NDA clause 7 issues in two weeks
    → memory_search("NDA clause 7") → finds Bob and Carol entries
    → Writes to memory/2026-02-23.md:
        "NDA clause 7 repeatedly causes confusion during onboarding.
         Proactively explain it in welcome emails."
    → Adds to MEMORY.md:
        "- NDA clause 7 causes confusion — proactively explain during onboarding"
    → Delivers to #hr channel: "I've noticed NDA clause 7 confuses
        new hires. I'll start explaining it proactively."

Week 3:
  Execution "Onboard Dave"
    → System prompt includes MEMORY.md content:
      "NDA clause 7 causes confusion — proactively explain during onboarding"
    → Agent includes clause 7 explanation in welcome email
    → Dave doesn't get stuck
    → Activity feed: "Onboard Dave: completed smoothly"
```

The agent learned from its own experience. No human intervention needed.

---

## System Prompt Integration

### MEMORY.md (Pinned Memories)

Loaded verbatim into every session's system prompt:

```
## Agent Memory
The following are facts you have learned and saved to MEMORY.md.
Refer to them as needed. You can update MEMORY.md at any time.

- NDA clause 7 causes confusion during onboarding — explain proactively
- Alice (VP Ops) prefers Slack, timezone US/Pacific
- Company fiscal year starts April 1
- Staging environment uses Let's Encrypt certs, renewed quarterly
```

Capped at ~4000 tokens. If `MEMORY.md` grows too large, the agent is
prompted to curate it (remove stale entries, consolidate duplicates).

### Memory Recall Guidance

Added to the system prompt for all sessions:

```
## Memory
You have a memory system with two tiers:
- MEMORY.md: your curated knowledge (already loaded above).
- memory/YYYY-MM-DD.md: daily logs (searchable via memory_search).

**Reading:** Use memory_search(query) before answering anything about prior
work, decisions, dates, people, preferences, or processes.

**Writing:** Use the write tool to save memories:
- Important/permanent facts → add to MEMORY.md
- Daily observations/notes → write to memory/YYYY-MM-DD.md
- To promote a fact → move it from daily log to MEMORY.md
- To forget → remove the line from the file

Create the memory/ directory if it doesn't exist.
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
Write observations to memory/YYYY-MM-DD.md. If you spot a pattern
important enough to always remember, add it to MEMORY.md.

If something needs action, create a task or deliver a message to
the relevant channel.

If nothing needs attention, reply HEARTBEAT_OK.
```

---

## Pre-Compaction Flush

Same pattern as OpenClaw, adapted for file-based writing.

### Trigger

When `total_tokens` crosses:
`context_window - reserve_floor - soft_threshold`

Default: fires when ~4000 tokens remain before compaction.

### Mechanism

1. Check: `memory_flush_compaction_count != compaction_count` (one flush
   per compaction cycle)
2. Run silent agent turn with:
   - System: "Session nearing compaction. Write durable facts to your
     memory files before context is compressed."
   - User: "Pre-compaction flush. Write important memories to MEMORY.md
     or memory/YYYY-MM-DD.md now. Reply NO_REPLY if nothing to save."
3. Agent writes to memory files using `write` tool
4. Post-turn sync persists changes to DB with embeddings
5. Update `memory_flush_compaction_count = compaction_count`
6. Compaction proceeds normally

### Which Sessions Get Flush

| Session Type | Pre-Compaction Flush |
|---|---|
| Channel (triage) | Yes — accumulates across weeks |
| Heartbeat | Yes — accumulates across all ticks |
| Execution (long-running) | Yes — multi-run executions with deep context |
| Scheduled (independent) | Yes — accumulates across fires |
| Scheduled (notify) | No — fires into parent session, flush happens there |

---

## Post-Run Sync Details

The sync runs after every agent turn that touches memory files. It bridges
the file-based writing surface with the DB-backed search index.

### Sync Flow

```python
def sync_memory_files(agent_id, org_id, session_id, sandbox_path):
    """Run after every agent turn. Idempotent."""

    # 1. Sync MEMORY.md → pinned entries
    memory_md = read_file(sandbox_path / "MEMORY.md")
    if memory_md:
        entries = parse_memory_entries(memory_md)  # split by line/block
        existing = db.query(
            "SELECT id, content, line_hash FROM memories "
            "WHERE agent_id = %s AND file_path = 'MEMORY.md'",
            agent_id
        )
        diff = compute_diff(existing, entries)
        for entry in diff.added:
            db.insert(memories, agent_id=agent_id, org_id=org_id,
                      content=entry, file_path="MEMORY.md",
                      line_hash=sha256(entry), pinned=True,
                      source_session_id=session_id)
        for entry in diff.removed:
            db.update(memories, id=entry.id, pinned=False)
        for entry in diff.changed:
            db.update(memories, id=entry.id, content=entry.new_content,
                      line_hash=sha256(entry.new_content))

    # 2. Sync memory/*.md → unpinned entries
    for file in glob(sandbox_path / "memory/*.md"):
        file_path = f"memory/{file.name}"
        content = read_file(file)
        entries = parse_memory_entries(content)
        existing = db.query(
            "SELECT id, content, line_hash FROM memories "
            "WHERE agent_id = %s AND file_path = %s",
            agent_id, file_path
        )
        diff = compute_diff(existing, entries)
        for entry in diff.added:
            db.insert(memories, agent_id=agent_id, org_id=org_id,
                      content=entry, file_path=file_path,
                      line_hash=sha256(entry), pinned=False,
                      source_session_id=session_id)
        for entry in diff.changed:
            db.update(memories, id=entry.id, content=entry.new_content,
                      line_hash=sha256(entry.new_content))

    # 3. Batch embed all new/changed entries
    pending = db.query(
        "SELECT id, content FROM memories "
        "WHERE agent_id = %s AND embedding IS NULL",
        agent_id
    )
    if pending:
        embeddings = batch_embed([e.content for e in pending])
        for entry, emb in zip(pending, embeddings):
            db.update(memories, id=entry.id, embedding=emb)
```

### Parse Strategy

`MEMORY.md` entries are parsed as individual bullet points or paragraph
blocks. `memory/*.md` entries are parsed as paragraph blocks (separated by
blank lines). This gives the agent flexibility in how it structures notes
while keeping each entry small enough for meaningful vector search.

### Sandbox File Seeding

On session start, the memory files are seeded into the sandbox from the DB:

1. Query pinned entries → write to `MEMORY.md`
2. Query recent unpinned entries for the current date → write to
   `memory/YYYY-MM-DD.md`

This means the agent always sees its latest memories as real files it can
read and edit. The files are the interface; the DB is the backing store.

---

## Component Map

| Component | File | Role |
|---|---|---|
| Memory model | `memory/models.py` | `Memory` SQLAlchemy table |
| Memory repository | `memory/repository.py` | CRUD + vector search |
| Memory service | `memory/service.py` | Business logic, embedding, search |
| Memory search tool | `agent/tools/memory.py` | 1 agent tool (search only) |
| Post-run sync | `memory/sync.py` | File → DB sync after agent turns |
| Sandbox seeder | `memory/seed.py` | DB → file seeding on session start |
| Activity feed model | `memory/models.py` | `ActivityFeed` table |
| Activity feed writer | `memory/activity.py` | System-generated event logging |
| Activity feed pruner | `memory/activity.py` | 14-day retention sweep |
| Pre-compaction flush | `worker/memory_flush.py` | Silent turn before compaction |
| Pinned memory loader | `agent/prompt.py` | Load MEMORY.md into system prompt |
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

    # Pinned (MEMORY.md)
    max_pinned_tokens: int = 4000

    # Activity feed
    activity_feed_enabled: bool = True
    activity_feed_retention_days: int = 14

    # Pre-compaction flush
    flush_enabled: bool = True
    flush_soft_threshold_tokens: int = 4000
    flush_reserve_floor_tokens: int = 20000

    # Post-run sync
    sync_enabled: bool = True
    sync_batch_embed_size: int = 50
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
