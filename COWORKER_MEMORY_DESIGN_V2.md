# Memory System Design v2
**Date:** 2026-02-23
**Status:** Draft

## Problem

The coworker platform runs isolated sessions — channel sessions per user,
execution sessions per task, heartbeat session per agent. No single session
sees everything. An execution that onboards Alice doesn't know that the
execution that onboarded Bob hit the same NDA issue last week.

The agent needs a memory system that:
1. Bridges session isolation so it learns across executions
2. Scopes knowledge per workflow so memories stay relevant
3. Captures human feedback on specific workflow steps
4. Requires zero special tools — writing is just writing to files

## Design Principles

- **Files are the interface.** The agent writes memories using the same
  `write`/`edit` tools it already has. No `save_memory`, `pin_memory`, or
  any memory-specific write tools.
- **Blob storage is the source of truth.** Memory files live in blob storage
  (S3/GCS/R2). Durable, cheap, inspectable. The DB is just a search index.
- **Workflow-scoped knowledge.** Like nested `CLAUDE.md` files in a monorepo,
  each workflow gets its own memory file. Execution agents see global +
  workflow-specific memories.
- **Heartbeat is the brain.** It reads the activity feed, spots patterns
  across executions, and writes memories. Executions are the hands — they
  do work and learn along the way.

---

## File Structure

```
/{org_id}/agents/{agent_id}/
  MEMORY.md                                 ← global pinned (always in prompt)
  memory/2026-02-17.md                      ← global daily log
  memory/2026-02-23.md                      ← global daily log
  workflows/{workflow_id}/MEMORY.md         ← workflow-specific pinned
  workflows/{workflow_id}/MEMORY.md         ← another workflow
```

### Three File Types

| File | Scope | In prompt | Searchable | Who writes |
|---|---|---|---|---|
| `MEMORY.md` | Global | Every session | Yes | Heartbeat, triage |
| `memory/YYYY-MM-DD.md` | Global | No | Yes | All sessions |
| `workflows/{id}/MEMORY.md` | Workflow | Workflow executions | Yes | Executions, heartbeat |

### The CLAUDE.md Analogy

Like a monorepo with nested `CLAUDE.md` files:

```
Root CLAUDE.md        →  MEMORY.md (global rules, agent-wide knowledge)
packages/foo/CLAUDE.md →  workflows/{id}/MEMORY.md (package-specific rules)
```

When you open a file in `packages/foo/`, your editor loads both the root
and the package CLAUDE.md. When the agent runs a workflow execution, it
loads both global MEMORY.md and the workflow's MEMORY.md.

---

## Storage Architecture

```
┌─────────────────────┐      ┌──────────────────────┐
│   Agent Sandbox      │      │   Blob Storage       │
│   (ephemeral)        │ sync │   (source of truth)  │
│                      │ ←──→ │                      │
│  MEMORY.md           │      │  MEMORY.md           │
│  memory/*.md         │      │  memory/*.md         │
│  workflows/X/MEM..  │      │  workflows/X/MEM..  │
└─────────────────────┘      └──────────┬───────────┘
                                        │
                                  async indexer
                                        │
                              ┌─────────↓──────────┐
                              │   Search Index      │
                              │   (pgvector DB)     │
                              │                     │
                              │   chunks + embeddings│
                              │   (derived, not     │
                              │    source of truth) │
                              └─────────────────────┘
```

**Blob storage** holds the canonical memory files. Cheap, durable, inspectable.

**Search index** (pgvector) holds chunked + embedded copies for
`memory_search`. Derived from blob — can be rebuilt at any time.

**Sandbox** is the agent's working copy. Seeded from blob on session start,
synced back to blob on turn end.

---

## What Each Session Sees

### Execution: Workflow Task

The most important case. The agent gets layered context:

```
System prompt:
  1. Global MEMORY.md         ← "Company fiscal year starts April 1"
  2. Workflow MEMORY.md       ← "NDA clause 7 causes confusion — explain proactively"
  3. Memory recall guidance   ← "Use memory_search for daily logs"
```

The agent's sandbox contains:
- `MEMORY.md` (global, read-only during execution)
- `workflows/{workflow_id}/MEMORY.md` (workflow-specific, writable)
- `memory/YYYY-MM-DD.md` (daily log, writable)

### Execution: Free-Form Task

No workflow scoping. Just global context:

```
System prompt:
  1. Global MEMORY.md
  2. Memory recall guidance
```

Sandbox contains:
- `MEMORY.md` (global, read-only during execution)
- `memory/YYYY-MM-DD.md` (daily log, writable)

### Heartbeat Session

Sees everything. Can write everywhere:

```
System prompt:
  1. Global MEMORY.md
  2. Activity feed (since last tick)
  3. Heartbeat instructions
```

Sandbox contains all memory files. Can write to global MEMORY.md, any
workflow MEMORY.md, and daily logs.

### Channel Session (Triage)

Global context + ability to write workflow feedback:

```
System prompt:
  1. Global MEMORY.md
  2. Memory recall guidance
```

Sandbox contains:
- `MEMORY.md` (global, writable)
- `memory/YYYY-MM-DD.md` (daily log, writable)
- Workflow MEMORY.md files loaded on-demand when user gives workflow feedback

---

## Workflow Memory In Detail

### Structure of a Workflow MEMORY.md

The agent organizes workflow knowledge using markdown headers by step:

```markdown
# Onboarding Workflow

## General
- Average onboarding takes 3 business days
- Always CC hiring manager on all emails
- Contractors follow a shorter flow (skip steps 4-6)

## Step: send-welcome-email
- Send BEFORE the NDA package, not after (feedback: Alice, 2026-02-20)
- Include team intro for engineering hires

## Step: send-documents
- Use short-form contract for contractors (feedback: Bob, 2026-02-23)
- Include IP assignment addendum for engineering hires

## Step: schedule-orientation
- Book 60min slots, 30min was consistently too short
- Prefer Tuesday/Thursday for new hire orientation
```

No step-level files. One file per workflow with headers for structure.
The agent reads the whole file, sees step-specific notes in context
with neighboring steps.

### How Feedback Gets Captured

When a user gives feedback on a workflow step through a channel:

```
User (Slack): "For onboarding, step 3 should send the welcome
               email before the NDA"

Triage agent:
  1. Recognizes workflow feedback
  2. Loads workflows/wf_onboarding/MEMORY.md into sandbox
  3. Appends under "## Step: send-documents":
     "- Send welcome email BEFORE the NDA (feedback: Alice, 2026-02-20)"
  4. Syncs back to blob
  5. Confirms: "Got it, I'll send the welcome email before the NDA
     in future onboarding runs."
```

Next time the onboarding workflow runs, the execution agent sees this
note in its prompt and follows it.

### How Executions Write Workflow Memory

During a workflow execution, the agent discovers something worth remembering:

```
Execution "Onboard Carol":
  → Carol got confused by NDA clause 7
  → Agent writes to workflows/wf_onboarding/MEMORY.md:
    Under "## Step: send-documents":
    "- NDA clause 7 needs extra explanation (discovered 2026-02-23)"
  → Agent writes to memory/2026-02-23.md:
    "Carol confused by NDA clause 7 during onboarding"
```

The workflow MEMORY.md captures the procedural fix. The daily log
captures the event for heartbeat visibility.

### How Heartbeat Promotes Patterns

The heartbeat spots a pattern across executions and promotes it:

```
Heartbeat tick:
  → Reads activity feed: two NDA clause 7 issues in two weeks
  → memory_search("NDA clause 7") → finds daily log entries
  → Writes to workflows/wf_onboarding/MEMORY.md under "## General":
    "- NDA clause 7 repeatedly confuses new hires — proactively
       explain it before sending documents"
  → Optionally promotes to global MEMORY.md if it applies beyond
    this workflow
```

---

## Daily Logs

One global daily log stream. All sessions write here.

```
# memory/2026-02-23.md

Onboard Carol: completed. NDA clause 7 caused confusion again.
She asked about IP assignment for her consulting arrangement.

Triage (Slack, Alice): asked about Q2 budget timeline. Answered
from memory — fiscal year starts April 1.

Deploy review wf_deploy_v2: staging deploy passed. Noted that
Let's Encrypt cert renewal is due next week.
```

Daily logs are not loaded into prompts. They're searchable via
`memory_search` and visible to the heartbeat via the activity feed.
They serve as the raw event stream that the heartbeat mines for patterns.

---

## Agent Tool

One memory-specific tool. For reading only:

```
memory_search(
    query: str,
    workflow_id: str | None = None,
    limit: int = 6
) -> [{ content, file_path, score, created_at }]
```

- No `workflow_id` → searches global memories (MEMORY.md + daily logs),
  excludes other workflows
- With `workflow_id` → searches global + that workflow's memories
- Hybrid: vector similarity (70%) + keyword BM25 (30%)

Writing uses the existing `write`/`edit` tools. The agent writes to files.
The system handles syncing files to blob and indexing for search.

---

## Activity Feed

Automatic, system-generated. No agent involvement in writing. Gives the
heartbeat cross-session visibility.

### Events

| Event | Trigger | Summary |
|---|---|---|
| `execution_completed` | Execution finishes | Last agent message or result |
| `execution_failed` | Execution errors | Error + task context |
| `task_status_changed` | Status transitions | "{title}: {old} → {new}" |
| `schedule_fired` | Scheduler fires | "{name} fired (run #{count})" |
| `channel_interaction` | Substantive triage msg | First sentence of reply |
| `feedback_received` | User gives workflow feedback | "{workflow}: {summary}" |

14-day rolling window. Auto-pruned. Short-term bridge, not permanent record.

---

## Pre-Compaction Flush

When any long-running session nears its context limit, a silent agent turn
fires before compaction. The agent writes durable facts to its memory files.
The system syncs changes to blob.

Applies to: channel sessions, heartbeat, long-running executions,
independent scheduled tasks. Does not apply to notify/resume sessions.

One flush per compaction cycle, tracked via counter on the session.

---

## Memory Strategy Per Session Type

### Channel Session (triage)

| Aspect | Strategy |
|---|---|
| **Prompt** | Global MEMORY.md |
| **Writes** | Daily log. Global MEMORY.md for user preferences. Workflow MEMORY.md when capturing feedback. |
| **Search** | Global scope by default. Workflow scope when user references a workflow. |

### Heartbeat Session

| Aspect | Strategy |
|---|---|
| **Prompt** | Global MEMORY.md + activity feed (since last tick) |
| **Writes** | Global MEMORY.md (cross-workflow patterns). Workflow MEMORY.md (workflow-specific patterns). Daily log. |
| **Search** | All scopes (global + all workflows). |
| **Actions** | Deliver messages, create tasks, schedule tasks. |

### Execution: Workflow Task

| Aspect | Strategy |
|---|---|
| **Prompt** | Global MEMORY.md + workflow MEMORY.md |
| **Writes** | Workflow MEMORY.md (step-level discoveries). Daily log (event record). |
| **Search** | Global + this workflow's memories. |

### Execution: Free-Form Task

| Aspect | Strategy |
|---|---|
| **Prompt** | Global MEMORY.md |
| **Writes** | Daily log. Global MEMORY.md if broadly relevant. |
| **Search** | Global scope only. |

### Scheduled Task (independent)

| Aspect | Strategy |
|---|---|
| **Prompt** | Global MEMORY.md (+ workflow MEMORY.md if workflow-linked) |
| **Writes** | Daily log. Cross-fire observations to relevant MEMORY.md. |
| **Search** | Global (+ workflow if linked). |

### Scheduled Task (notify/resume)

| Aspect | Strategy |
|---|---|
| **Memory** | Inherits from target session. |

---

## The Value Loop

```
                   ┌──────────────────────────────┐
                   │     HEARTBEAT SESSION         │
                   │  (the brain)                 │
                   │                               │
                   │  Reads: activity feed         │
                   │  Reads: memory_search (all)   │
                   │  Writes: MEMORY.md            │
                   │  Writes: workflows/*/MEMORY.md│
                   │  Acts: deliver, create task   │
                   └──────────┬────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │ writes          │ reads            │ acts
            ↓                 ↓                  ↓
    ┌───────────────┐ ┌──────────────┐  ┌──────────────┐
    │  BLOB STORAGE │ │ SEARCH INDEX │  │  TASK BOARD  │
    │  (files)      │ │ (pgvector)   │  │  (new work)  │
    └───────┬───────┘ └──────────────┘  └──────┬───────┘
            │                                   │
            │ seeded into sandbox                │ worker picks up
            ↓                                   ↓
    ┌───────────────┐                   ┌──────────────┐
    │  EXECUTION    │                   │  EXECUTION   │
    │  (workflow)   │                   │  (free-form) │
    │               │                   │              │
    │  Reads:       │                   │  Reads:      │
    │   MEMORY.md   │                   │   MEMORY.md  │
    │   wf/MEMORY.md│                   │              │
    │  Writes:      │                   │  Writes:     │
    │   wf/MEMORY.md│                   │   daily log  │
    │   daily log   │                   │              │
    └───────┬───────┘                   └──────┬───────┘
            │                                   │
            │ generates                         │ generates
            ↓                                   ↓
    ┌─────────────────────────────────────────────────┐
    │              ACTIVITY FEED                       │
    │  (automatic, 14-day window)                     │
    └──────────────────────┬──────────────────────────┘
                           │ heartbeat reads
                           ↓
                   ┌───────────────────────┐
                   │   HEARTBEAT (next)    │
                   └───────────────────────┘
```

---

## Concrete Example: Learning Loop

```
Week 1:
  Execution "Onboard Bob" (workflow: wf_onboarding)
    → Bob had questions about NDA clause 7
    → Agent writes to workflows/wf_onboarding/MEMORY.md:
        ## Step: send-documents
        - NDA clause 7 needs extra explanation (2026-02-17)
    → Agent writes to memory/2026-02-17.md:
        "Bob struggled with NDA clause 7 during onboarding"
    → Activity feed: "Onboard Bob: completed, NDA clause 7 confusion"

Week 2:
  Execution "Onboard Carol" (workflow: wf_onboarding)
    → System prompt already includes workflow MEMORY.md with the
      clause 7 note from Bob's onboarding
    → Agent explains clause 7 proactively, but Carol still has
      follow-up questions about IP assignment
    → Agent appends to workflows/wf_onboarding/MEMORY.md:
        ## Step: send-documents
        - Also explain IP assignment clause for consulting hires
    → Activity feed: "Onboard Carol: completed, IP clause questions"

  User feedback (Slack, Alice):
    "For onboarding, always send the welcome email before the NDA"
    → Triage agent writes to workflows/wf_onboarding/MEMORY.md:
        ## Step: send-welcome-email
        - Send BEFORE the NDA package (feedback: Alice, 2026-02-23)

  Heartbeat tick:
    → Reads activity feed: two onboarding runs with document confusion
    → memory_search("onboarding documents") → finds both entries
    → Writes to workflows/wf_onboarding/MEMORY.md under ## General:
        "- Document step consistently causes confusion. Consider
           adding a FAQ section to the welcome email."
    → Delivers to #hr: "I've improved the onboarding document step
        based on recent runs. Clause 7 and IP assignment are now
        explained proactively."

Week 3:
  Execution "Onboard Dave" (workflow: wf_onboarding)
    → System prompt includes:
        Global MEMORY.md: company-wide facts
        Workflow MEMORY.md: all accumulated onboarding knowledge
          - Send welcome email before NDA
          - Explain clause 7 proactively
          - Explain IP assignment for consulting hires
          - Consider adding FAQ section
    → Agent follows all the learned procedures
    → Dave's onboarding goes smoothly
    → Activity feed: "Onboard Dave: completed smoothly"
```

---

## System Prompt Templates

### Global Memory Section (all sessions)

```
## Agent Memory
{contents of MEMORY.md}
```

### Workflow Memory Section (workflow executions only)

```
## Workflow Memory: {workflow_name}
{contents of workflows/{workflow_id}/MEMORY.md}
```

### Memory Recall Guidance (all sessions)

```
## Memory
You have a layered memory system:
- MEMORY.md: global knowledge (loaded above)
- workflows/{id}/MEMORY.md: workflow-specific knowledge (loaded above
  when running a workflow)
- memory/YYYY-MM-DD.md: daily logs (searchable via memory_search)

**Reading:** Use memory_search(query) before answering anything about
prior work, decisions, people, or processes.

**Writing:** Use the write tool:
- Workflow-specific facts → workflows/{id}/MEMORY.md (use step headers)
- General facts about people/org → MEMORY.md
- Daily observations → memory/YYYY-MM-DD.md
```

### Heartbeat Instructions

```
## Recent Activity (since last heartbeat)
{activity feed entries}

## Instructions
Review the activity above. Look for:
- Patterns across executions (repeated issues, common blockers)
- Workflow steps that consistently cause problems
- Overdue or stalled tasks
- Opportunities to improve processes

Use memory_search for deeper context. Write observations:
- Cross-workflow patterns → MEMORY.md
- Workflow-specific patterns → workflows/{id}/MEMORY.md
- Daily notes → memory/YYYY-MM-DD.md

If something needs action, create a task or deliver a message.
If nothing needs attention, reply HEARTBEAT_OK.
```

---

## Search Index

The DB holds chunked + embedded copies of memory file content. It exists
solely to power `memory_search`. It is not the source of truth — blob
storage is.

### Schema

```
memory_index
┌──────────────────────────────────────────────────────────────┐
│ id                 UUID PK                                   │
│ org_id             UUID FK                                   │
│ agent_id           UUID FK                                   │
│ file_path          VARCHAR(255) NOT NULL                      │
│ chunk_index        INT NOT NULL                               │
│ chunk_text         TEXT NOT NULL                              │
│ embedding          vector(1536)                               │
│ indexed_at         TIMESTAMPTZ NOT NULL                       │
│                                                              │
│ Indexes:                                                     │
│   (agent_id, file_path, chunk_index) UNIQUE                  │
│   HNSW on embedding                                          │
└──────────────────────────────────────────────────────────────┘
```

`file_path` determines scope:
- `MEMORY.md` → global pinned
- `memory/2026-02-23.md` → global daily
- `workflows/wf_onboarding/MEMORY.md` → workflow pinned

Search scoping uses `WHERE file_path` filters. No separate `pinned`
column — the path tells you.

---

## Concurrency

Low risk by design:

- **Global MEMORY.md**: written by heartbeat (runs on a timer, serialized)
  and triage (rare writes). Conflict window is small.
- **Workflow MEMORY.md**: written by that workflow's executions (serialized
  per the task concurrency guard) and heartbeat. Low contention.
- **Daily logs**: append-only. Multiple sessions may write to the same
  date file, but appends don't conflict.
- **Blob storage**: last-write-wins for MEMORY.md files. For the rare
  conflict, ETag-based optimistic locking (conditional PUT) can retry
  with a merge.

---

## Future Considerations (Not In Scope)

- Memory sharing across agents within the same org
- Confidence decay / staleness scoring
- Memory deduplication (near-duplicate detection)
- Memory export / import for agent migration
- RAG over external documents (company knowledge base)
- Workflow step-level files (if single-file-per-workflow proves insufficient)
