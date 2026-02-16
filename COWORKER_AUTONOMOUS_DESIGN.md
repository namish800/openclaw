# Digital Coworker — Autonomous Worker Design

## Overview

The digital coworker is not a chatbot or helpdesk. It IS an employee — it has a role, a job title, responsibilities, ongoing relationships, and a workday. People interact with it like they would a colleague: they email it, CC it, mention it in Slack, invite it to meetings.

This document covers everything above the gateway layer: the "brain" that makes the coworker an autonomous worker rather than a reactive message processor.

---

## Architecture Stack

```
┌──────────────────────────────────────────┐
│            COMMUNICATION LAYER           │
│  Gateway (email, Slack, Teams, etc.)     │
│  Inbound pipeline, outbound delivery     │
│  See: COWORKER_GATEWAY_DESIGN.md         │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│            EXECUTION LAYER               │
│  AgentInvoker + LLM + Tools             │
│  Sub-agents for parallel/background work │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│            BRAIN LAYER                   │
│  Task queue (what to do)                 │
│  Scheduler (when to do it)              │
│  Workflow engine (multi-step processes)  │
│  Memory (what I've learned)             │
│  Guardrails (what I'm NOT allowed to do)│
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│            IDENTITY LAYER                │
│  Persona (SOUL — who am I)              │
│  Operating rules (AGENTS — how I work)  │
│  Tool config (what systems I can access) │
│  Org context (company, team, policies)   │
└──────────────┬───────────────────────────┘
               │
┌──────────────▼───────────────────────────┐
│            OBSERVABILITY LAYER           │
│  Audit log (what I did and why)         │
│  Activity dashboard (status, metrics)    │
│  Escalation tracking                     │
│  Cost monitoring (tokens, API calls)     │
└──────────────────────────────────────────┘
```

---

## Proactive Engine

The coworker has four mechanisms for proactive behavior. These drive it to act without waiting for inbound messages.

### 1. Heartbeat (Periodic Self-Check)

Periodic agent wake-up that checks for pending work. The safety net — if a scheduled task was missed, if an event trigger failed, the heartbeat catches it.

**Inspired by:** OpenClaw's heartbeat system (30-minute interval, HEARTBEAT.md checklist, HEARTBEAT_OK silent dismissal).

**How it works:**
```
Heartbeat fires (configurable interval per org, default 30 min)
    │
    ├── Pre-flight: within active hours? Any ready tasks in queue?
    │   (skip if no tasks — saves LLM cost)
    │
    ├── Query task queue for ready tasks
    │
    ├── For each task (priority order):
    │   ├── Load session context
    │   ├── Invoke agent with task prompt + session history
    │   ├── Agent acts (sends emails, updates systems, etc.)
    │   ├── Mark task complete
    │   └── Any new tasks created? Add to queue
    │
    └── Done. Sleep until next heartbeat.
```

**Key differences from OpenClaw:**
- OpenClaw: single agent, reads a file (HEARTBEAT.md), one session
- Coworker: per-org, queries DB for pending tasks, multiple sessions
- OpenClaw: always invokes LLM (agent reads file and decides)
- Coworker: skip LLM entirely if no pending tasks (cheaper)

**Configuration (per org):**
```python
class HeartbeatConfig:
    enabled: bool = True
    interval_minutes: int = 30
    active_hours_start: str | None = None    # "08:00"
    active_hours_end: str | None = None      # "22:00"
    active_hours_timezone: str | None = None # "America/New_York"
    max_tasks_per_beat: int = 10             # Don't process more than N tasks per heartbeat
```

### 2. Scheduler (Time-Based Triggers)

Exact timing for recurring work and one-shot reminders.

**Inspired by:** OpenClaw's cron system (5-field cron expressions, one-shot `--at`, isolated vs main session).

**Two types:**

**Recurring (cron):**
- "Every morning at 9am, check CRM for new leads and send outreach"
- "Every Friday at 4pm, send payment reminders for overdue invoices"
- "Every Monday, generate weekly pipeline report"

**One-shot (delayed):**
- "Follow up with Jane in 3 days"
- "Retry calling John in 30 minutes"
- "Send renewal notice 60 days before contract expires"

**Session handling:**

| Type | Session | Why |
|---|---|---|
| Recurring reports | Fresh/isolated | Standalone task, no conversation context needed |
| Relationship follow-ups | Resume existing session | Agent needs thread history to write a good follow-up |
| Deadline checks | Fresh, queries DB | Checks across all sessions, doesn't belong to one |

**Data model:**
```python
class ScheduledTask:
    id: str
    org_id: str
    kind: str                        # "cron" | "one_shot"

    # When
    cron_expr: str | None            # Recurring: "0 9 * * MON-FRI"
    run_at: datetime | None          # One-shot: exact time
    timezone: str                    # IANA timezone

    # What
    prompt: str                      # What to tell the agent
    context: dict[str, Any] | None   # Extra context

    # Where
    session_id: str | None           # None = fresh session, set = resume existing
    channel_type: str | None         # Where to deliver output
    deliver_to: str | None           # Recipient

    # Lifecycle
    delete_after_run: bool = False   # True for one-shots
    max_attempts: int = 3
    created_by_session: str | None   # Which session created this
    created_at: datetime
    last_run_at: datetime | None
```

### 3. Event Triggers (External System Webhooks)

External business systems poke the coworker when something happens.

**Inspired by:** OpenClaw's `/hooks/wake` webhook endpoint and Gmail Pub/Sub integration.

**Examples:**

| Source | Event | Coworker Action |
|---|---|---|
| CRM | New lead assigned | Start outreach sequence |
| CRM | Deal stage changed | Send appropriate follow-up |
| HR system | New employee added | Start onboarding workflow |
| HR system | PTO approved | Send confirmation to employee |
| Finance | Invoice received | Start processing workflow |
| Finance | Payment cleared | Send confirmation to vendor |
| Monitoring | Error rate spiked | Create incident, notify on-call |
| Calendar | Contract expiring in 30 days | Initiate renewal conversation |

**Flow:**
```
External system webhook → coworker_api endpoint
    │
    ├── Validate webhook (signature, source)
    ├── Map event to task template
    ├── Create task in queue (or invoke agent immediately)
    └── Return 200
```

**Session handling:**
- New workflow (new lead) → create new session
- Continuing workflow (payment cleared for existing invoice) → resume session via correlation
- Notification only (deployment done) → no session, fire-and-forget outbound

### 4. Task Queue (Deferred Work Within a Session)

During an agent run, the agent can create tasks for later execution. This is the coworker's internal to-do list.

**Inspired by:** OpenClaw's followup queue (in-memory, per-session deferred work).

**Key difference from OpenClaw:** Durable (DB-backed), survives restarts, has priority and dependencies.

**How tasks enter the queue:**

| Source | Example |
|---|---|
| Inbound message | Jane emails → task: "Respond to Jane" |
| Scheduler | Cron fires → task: "Send daily digest" |
| Workflow side-effect | While handling email, agent creates task: "Ask manager for approval" |
| External webhook | CRM event → task: "Start outreach to new lead" |
| Agent tool call | Agent calls `schedule_task()` during execution |

**Data model:**
```python
class Task:
    id: str
    org_id: str

    # What
    description: str                    # Human-readable: "Follow up with Acme Corp"
    prompt: str                         # What to tell the agent when executing
    context: dict[str, Any] | None      # Extra context (workflow step, attempt count)

    # Where (session linkage)
    session_id: str | None              # Existing session to resume (None = create new)
    channel_type: str | None            # Where to deliver output
    deliver_to: str | None              # Recipient

    # When
    status: TaskStatus                  # ready, blocked, in_progress, completed, failed
    ready_at: datetime | None           # Don't execute before this time
    due_at: datetime | None             # SLA deadline

    # Priority
    priority: int                       # 0 = highest, 100 = lowest

    # Lineage
    source: TaskSource                  # inbound_message, scheduler, workflow, webhook
    parent_task_id: str | None          # Task that spawned this one
    blocking_task_id: str | None        # Task this is waiting on

    # Lifecycle
    created_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    attempt: int = 0
    max_attempts: int = 3


class TaskStatus(str, Enum):
    READY = "ready"                     # Can be picked up
    BLOCKED = "blocked"                 # Waiting on time, dependency, or external response
    IN_PROGRESS = "in_progress"         # Agent is working on it now
    COMPLETED = "completed"
    FAILED = "failed"


class TaskSource(str, Enum):
    INBOUND_MESSAGE = "inbound"         # Someone messaged the coworker
    SCHEDULER = "scheduler"             # Cron or one-shot timer
    WORKFLOW = "workflow"               # Side-effect of another task
    WEBHOOK = "webhook"                 # External system event
```

**Execution flow:**
```
Heartbeat or immediate trigger
    │
    ├── Query: SELECT * FROM tasks WHERE org_id = ? AND status = 'ready'
    │          AND (ready_at IS NULL OR ready_at <= now())
    │          ORDER BY priority ASC, created_at ASC
    │
    ├── For each task:
    │   ├── Set status = in_progress
    │   ├── Load session (existing or create new)
    │   ├── Build prompt: task.prompt + task.context + session history
    │   ├── Call AgentInvoker.invoke()
    │   ├── Process agent output (send messages, update systems)
    │   ├── If agent creates new tasks → insert into queue
    │   ├── Mark task completed (or failed if error)
    │   └── If task has dependents → check if they're now unblocked
    │
    └── Done
```

**Tasks vs sessions:**

Tasks are **short-lived work items**. Sessions are **long-lived conversation contexts**. A task operates within a session.

```
Session: email:acme:deal-q1 (lives for weeks)
├── Task: "Send initial outreach" (completed, week 1)
├── Task: "Follow up - no reply" (completed, week 2)
├── Task: "Reply to Acme's questions" (completed, week 2)
├── Task: "Send proposal" (completed, week 3)
├── Task: "Follow up on proposal" (ready, due week 4) ← next heartbeat
└── ... session lives until deal closes or goes cold
```

Session lifetime is tied to the **relationship/workflow**, not a clock. No daily reset. A session closes when the workflow completes, it goes stale, or someone explicitly closes it.

---

## Identity Layer

How the coworker knows who it is and how to behave. Inspired by OpenClaw's bootstrap files (SOUL.md, AGENTS.md, IDENTITY.md, USER.md, TOOLS.md).

### Agent Persona (SOUL.md equivalent)

Defines WHO the coworker is — personality, tone, boundaries.

**OpenClaw:** Plain markdown file in the workspace, agent can modify it.
**Coworker:** Stored in DB, admin-configured via web UI, read-only for the agent.

```python
class AgentPersona:
    org_id: str
    name: str                        # "Alex"
    role: str                        # "HR Coordinator"
    tone: str                        # "Professional but warm, concise, action-oriented"
    boundaries: list[str]            # ["Never share salary info",
                                     #  "Always CC HR on escalations",
                                     #  "Don't commit to deadlines without checking"]
    communication_style: dict[str, str]  # Per-channel style overrides
                                         # {"email": "Formal, use full sentences",
                                         #  "slack": "Casual, use emoji sparingly"}
    signature: str | None            # Email signature block
```

The agent should NOT modify its own persona in a business context — that's an admin function. Unlike OpenClaw where the agent evolves its identity, a business coworker's persona is controlled by the organization.

### Operating Rules (AGENTS.md equivalent)

Defines HOW the coworker operates — procedures, safety rules, escalation policy.

**OpenClaw:** Markdown file with memory strategy, group chat rules, safety guidelines.
**Coworker:** Structured rules in DB, admin-configured.

```python
class AgentOperatingRules:
    org_id: str
    safety_rules: list[str]          # ["Never approve expenses over $500 without manager",
                                     #  "Never send external emails without checking allowlist"]
    escalation_policy: str           # "If unsure, escalate to admin@company.com"
    working_hours: str | None        # "9am-6pm ET, Monday-Friday"
    response_sla: dict[str, int]     # {"email": 3600, "slack": 300} (seconds)
    channel_behavior: dict[str, str] # Per-channel operating notes
    custom_instructions: str | None  # Free-form org-specific instructions
```

### Agent Profile (IDENTITY.md equivalent)

Public-facing identity metadata.

```python
class AgentProfile:
    org_id: str
    display_name: str                # "Alex from HR"
    email_address: str               # "alex@company.com"
    avatar_url: str | None
    email_signature: str | None      # HTML signature for outbound emails
    timezone: str                    # "America/New_York"
```

### Tool Configuration (TOOLS.md equivalent)

What systems the coworker has access to. Per-org, admin-configured.

```python
class AgentToolConfig:
    org_id: str
    available_tools: list[str]       # ["email", "calendar", "crm", "leave_system"]
    tool_credentials: dict[str, Any] # API keys/tokens (encrypted at rest)
    tool_notes: dict[str, str]       # Per-tool usage guidelines
                                     # {"crm": "Use HubSpot API v3",
                                     #  "calendar": "Book rooms via Google Calendar"}
```

### System Prompt Assembly

All identity pieces are assembled into the system prompt at invocation time:

```python
def build_system_prompt(
    org_id: str,
    message: NormalizedMessage | None,  # None for heartbeat/scheduled tasks
    task: Task | None,                  # None for reactive inbound
) -> str:
    persona = db.get_agent_persona(org_id)
    rules = db.get_operating_rules(org_id)
    tools = db.get_tool_config(org_id)
    org = db.get_organization(org_id)

    prompt = f"""You are {persona.name}, {persona.role} at {org.name}.

## Personality
{persona.tone}

## Boundaries
{format_list(persona.boundaries)}

## Operating Rules
{format_list(rules.safety_rules)}

## Escalation
{rules.escalation_policy}

## Current Time
{datetime.now(tz=persona.timezone).isoformat()} ({persona.timezone})
"""

    if message:
        channel = message.sender.channel_type
        prompt += f"""
## Channel Context
You are communicating via {channel}.
{persona.communication_style.get(channel, '')}
{rules.channel_behavior.get(channel, '')}
"""

    if task:
        prompt += f"""
## Current Task
{task.description}
Priority: {task.priority}
{f'Due: {task.due_at.isoformat()}' if task.due_at else ''}
"""

    prompt += f"""
## Available Tools
{format_list(tools.available_tools)}
"""

    return prompt
```

---

## Memory System

How the coworker remembers things across sessions and gets better over time.

**Inspired by:** OpenClaw's three-layer memory (MEMORY.md, daily logs, vector search).

### Memory Types

```
Memory Store (per org):
├── Relationship memory    "Acme Corp: prefers PDF invoices, main contact is Sarah, timezone PST"
├── Preference memory      "Jane likes bullet points, not paragraphs"
├── Procedural memory      "For this org, expenses over $500 need VP approval"
├── Outcome memory         "Cold emails sent Monday morning get 40% reply rate vs 25% on Friday"
├── Fact memory            "Company holiday Dec 23-Jan 2, office reopens Jan 3"
└── Interaction memory     "Last talked to vendor X on Jan 15, they were unhappy about delivery"
```

### Data Model

```python
class MemoryEntry:
    id: str
    org_id: str
    category: str                    # "relationship", "preference", "procedural", "fact", "outcome"
    subject: str                     # Who/what this is about ("Acme Corp", "Jane", "expense policy")
    content: str                     # The memory itself
    source_session_id: str | None    # Which session created this memory
    source_task_id: str | None       # Which task created this memory
    confidence: float                # 0.0-1.0, decays over time or gets reinforced
    created_at: datetime
    updated_at: datetime
    expires_at: datetime | None      # Some memories are time-bound
    embedding: list[float] | None    # Vector embedding for semantic search
```

### Agent Tools

The agent has two memory tools available during execution:

**memory_search(query, category?, limit?):**
Semantic search across memory entries. Returns relevant memories for the current context.

**memory_write(subject, content, category):**
Create or update a memory entry. Agent calls this when it learns something useful.

### Memory Lifecycle

1. **Creation:** Agent learns something during a conversation and calls `memory_write()`
2. **Retrieval:** Before each agent invocation, relevant memories are searched and injected into context
3. **Reinforcement:** If a memory is confirmed ("Yes, Acme does prefer PDFs"), confidence increases
4. **Decay:** Old memories with low confidence are deprioritized
5. **Cleanup:** Expired or superseded memories are archived

### Pre-Compaction Memory Flush

Before compacting a long session (summarizing older messages), the agent is prompted to persist important facts to memory. This prevents knowledge loss during compaction.

**Inspired by:** OpenClaw's auto-memory flush (silent turn before compaction to write durable notes).

---

## Context Window Management

Long-running sessions (3-week email thread with a prospect) fill up the context window.

**Inspired by:** OpenClaw's compaction system (auto-summarize older messages, manual `/compact`).

### Auto-Compaction

When session transcript exceeds a token threshold:
1. **Memory flush:** Agent writes important facts to memory store
2. **Summarize:** Older turns are summarized into a compact representation
3. **Keep recent:** Last N turns remain verbatim
4. **Inject summary:** Summary becomes the first message in the compacted session

### Email-Specific Optimization

For email threads, full transcript compaction may not be needed. Instead:
- Always include the latest email verbatim
- Summarize the thread history (not individual turns)
- Include relevant memories about the contact
- Include the task context

---

## Escalation System

When the coworker can't or shouldn't handle something alone.

### When to Escalate

| Trigger | Example |
|---|---|
| Authority boundary | "I can't approve expenses over $500" |
| Low confidence | "I'm not sure if this contract clause is standard" |
| Emotional situation | "Customer is angry and threatening legal action" |
| Explicit request | "Let me talk to a real person" |
| Policy violation detected | "This expense report has duplicate entries" |
| Repeated failure | "Third attempt to reach John, no response" |

### Escalation Flow

```
Agent determines escalation needed
    │
    ├── Identify recipient (from operating rules or org directory)
    ├── Package context:
    │   ├── Conversation summary
    │   ├── What was tried
    │   ├── Why escalating
    │   └── Recommendation (if any)
    │
    ├── Send via appropriate channel (Slack DM, email, etc.)
    ├── Create escalation record
    ├── Notify the original requester ("I've looped in Sarah from HR")
    │
    └── Track:
        ├── Did the human acknowledge?
        ├── If no response in X hours, remind them
        └── When resolved, update task and notify requester
```

### Data Model

```python
class Escalation:
    id: str
    org_id: str
    task_id: str                     # The task being escalated
    session_id: str                  # The conversation context
    reason: str                      # Why escalating
    escalated_to: str                # User ID of the human
    channel: str                     # How they were notified
    context_summary: str             # Packaged context
    recommendation: str | None       # Agent's suggestion
    status: str                      # "pending", "acknowledged", "resolved", "timed_out"
    created_at: datetime
    acknowledged_at: datetime | None
    resolved_at: datetime | None
    resolution: str | None           # What the human decided
    follow_up_at: datetime | None    # When to remind if no response
```

---

## Sub-Agents / Delegation

The coworker can spawn background workers for parallel or specialized tasks.

**Inspired by:** OpenClaw's `sessions_spawn` tool (isolated sub-agent runs with auto-announce).

### Use Cases

**Parallel research:**
SDR preparing for 5 sales calls → spawn 5 sub-agents to research each prospect in parallel → synthesize results.

**Specialized work:**
Main coworker handles email → kicks off sub-agent with a different model to generate a financial report (cheaper model, focused tools).

**Long-running tasks:**
Generate a 20-page QBR report → spawn sub-agent → main coworker continues handling other tasks → gets notified when report is ready.

### Implementation

Sub-agents are separate `AgentInvoker` calls with:
- Their own session (isolated, not shared with parent)
- Potentially different/cheaper model
- Scoped tool access (only tools needed for the subtask)
- Results posted back to parent task when complete
- Configurable timeout

---

## Guardrails / Safety Boundaries

What the coworker is NOT allowed to do. Enforced at both prompt level (agent is told the rules) and tool level (tools check permissions before executing).

### Action Limits

```python
class ActionLimits:
    org_id: str
    max_emails_per_day: int = 50            # Prevent spam
    max_expense_auto_approve: float = 100   # Auto-approve under this, escalate above
    require_human_approval_for: list[str]   # ["first_external_email", "purchase", "contract"]
    read_only_systems: list[str]            # ["competitor_data", "salary_db"]
    no_send_windows: list[dict]             # [{"day": "sunday", "start": "00:00", "end": "23:59"}]
```

### Content Guardrails

Rules injected into the system prompt and enforced by the operating rules:
- Never share salary or compensation information
- Never commit to pricing without approval
- Never make legal claims or binding promises
- Always include disclaimer in external communications (if configured)
- Never forward internal emails externally without approval

### Approval Gates

Certain actions require human approval before execution:
- Sending to a customer/external contact for the first time
- Spending money (booking, purchasing, subscribing)
- Accessing sensitive data (medical records, financial data)
- Modifying production systems
- Sending bulk communications (>5 recipients)

The agent pauses the workflow, creates an approval request, and resumes when approved.

---

## Observability Layer

How the organization sees what the coworker is doing.

### Activity Log

Every significant action is logged:

```python
class ActivityLogEntry:
    id: str
    org_id: str
    timestamp: datetime
    actor: str                       # "agent:alex" or "user:jane@company.com"
    action: str                      # "email_sent", "task_completed", "escalation_created"
    task_id: str | None
    session_id: str | None
    details: dict[str, Any]          # Action-specific data
    tokens_used: int | None          # LLM cost tracking
```

### Dashboard Metrics

| Metric | Description |
|---|---|
| Active tasks | How many tasks in ready/in_progress |
| Tasks completed today/week | Throughput |
| Average response time | How fast the coworker responds to inbound |
| Escalation rate | What percentage of tasks get escalated |
| Messages sent/received | Communication volume |
| Token usage | LLM cost per day/week/month |
| Error rate | Failed tasks as percentage |
| Active sessions | How many ongoing conversations |

### Audit Trail

For compliance and debugging:
- Full session transcripts (what the agent saw and said)
- System prompts used for each invocation
- Tool calls made and their results
- Decisions made and the reasoning (if thinking/reasoning is enabled)
- Approval requests and their outcomes

---

## Business Scenarios

### The Coworker as Executive Assistant

**Persona:** "Sarah's EA — manages calendar, drafts emails, tracks action items"

**Daily workflow:**
- Morning heartbeat: check calendar for today's meetings, prepare agendas
- Throughout day: screen inbound emails, auto-respond to routine ones, flag important ones
- Meeting follow-up: receive forwarded notes, extract action items, send to responsible parties
- End of day: send tomorrow's schedule summary

**Tools needed:** Email, Calendar, Document generation

### The Coworker as SDR

**Persona:** "Alex — qualifies leads, sends outreach, schedules demos"

**Daily workflow:**
- Morning cron: check CRM for new leads, start outreach sequences
- Throughout day: handle prospect replies, answer questions, share collateral
- Afternoon: follow up on stale threads (3-day, 7-day cadence)
- Weekly: update pipeline report, send to sales manager

**Tools needed:** Email, CRM, Calendar, Web search

### The Coworker as Customer Success Manager

**Persona:** "Jordan — onboards customers, monitors engagement, handles renewals"

**Daily workflow:**
- Check product usage dashboards for engagement drops
- Send onboarding sequences to new customers
- Handle support questions, escalate complex issues
- Track renewal dates, initiate conversations 60 days out

**Tools needed:** Email, CRM, Product analytics API, Knowledge base

### The Coworker as Recruiter

**Persona:** "Morgan — screens candidates, coordinates interviews, manages pipeline"

**Daily workflow:**
- Screen new applications against job requirements
- Send outreach to sourced candidates
- Coordinate interview schedules between candidates and panels
- Collect feedback, compile scorecards
- Send status updates and offer/rejection letters

**Tools needed:** Email, Calendar, ATS (applicant tracking system), Document generation

### The Coworker as Accounts Payable Clerk

**Persona:** "Casey — processes invoices, tracks payments, chases overdue"

**Daily workflow:**
- Process incoming invoices (extract data, match to POs)
- Route for approval based on amount thresholds
- Send payment confirmations to vendors
- Chase overdue payments (escalating tone over time)
- Weekly: generate AP aging report

**Tools needed:** Email, Finance system, Document extraction

### The Coworker as Operations Coordinator

**Persona:** "Riley — coordinates between departments, tracks projects, manages vendors"

**Daily workflow:**
- Track project milestones, send status updates
- Coordinate between teams ("engineering needs specs from marketing by Thursday")
- Manage vendor communications (RFQs, quotes, orders)
- New employee onboarding logistics (IT setup, badge, desk, orientation)

**Tools needed:** Email, Project management, Calendar, Procurement system

---

## Interaction Patterns

All business scenarios decompose into five patterns:

### Pattern 1: Question & Answer (Stateless)

Employee asks → agent retrieves from knowledge base → responds.

**Session:** Exists for follow-ups but no workflow state.
**Task:** Single task, completed immediately.
**Example:** "What's our parental leave policy?"

### Pattern 2: Single-Action Task

Employee requests → agent validates → executes via tool → confirms.

**Session:** Short-lived, closed on completion.
**Task:** Single task with tool call.
**Example:** "Book room A for tomorrow 2-3pm."

### Pattern 3: Approval Chain (Multi-Party, Async)

Employee requests → agent sends to approver → waits → processes result → may escalate → confirms.

**Session:** Long-lived, spans days. Multiple parties involved.
**Tasks:** Chain of dependent tasks (request → approval → execution → confirmation).
**Example:** Leave request, expense reimbursement, software access.

**This is the hard pattern.** Requires:
- Outbound to a different person (approver)
- Correlation when approver responds
- Workflow state persistence across days
- Timeout/escalation if no response

### Pattern 4: Proactive Scheduled Task

Cron trigger → agent checks conditions → acts if needed → delivers to affected people.

**Session:** Fresh/isolated per run.
**Task:** Created by scheduler.
**Example:** Daily digest, timesheet reminders, pipeline report.

### Pattern 5: Workflow Retry / Callback

Agent hits blocker → schedules retry → resumes in same session later.

**Session:** Existing session, resumed on retry.
**Task:** One-shot delayed task with `session_id` set.
**Example:** Call not answered → retry in 30 min. Document not received → check again tomorrow.

---

## Session Lifetime

Sessions for a digital coworker are fundamentally different from OpenClaw's model.

**OpenClaw:** Daily reset at 4am, or idle timeout after 60 minutes. Designed for single-user chat.

**Coworker:** Session lifetime is tied to the relationship or workflow, not a clock.

| Session Type | Lifetime | Closes When |
|---|---|---|
| Q&A | Minutes | Question answered, no follow-up |
| Single action | Minutes | Action completed |
| Approval chain | Days to weeks | Workflow completes (approved/denied) |
| Sales outreach | Weeks to months | Deal closes, lead goes cold, or prospect unsubscribes |
| Customer relationship | Months | Customer churns or account closes |
| Recurring report | Never (fresh each run) | Each run is a new session |

**Auto-close policy:** Sessions with no activity for N days (configurable, default 30) are marked stale. Agent can send a final "closing this thread" message or just archive silently.

---

## What Lives Where

```
packages/gateway/                    # Communication layer
  └── Messaging infrastructure
      Inbound pipeline, outbound delivery
      Channel plugins (email, Slack, etc.)
      Does NOT know about tasks, workflows, or memory

packages/coworker_api/               # Everything else
  ├── gateway/                       # Gateway integration (invoker, session resolver, etc.)
  │
  ├── brain/                         # The autonomous worker engine
  │   ├── tasks/                     # Task queue, executor, priority
  │   ├── scheduler/                 # Cron + one-shot timers
  │   ├── heartbeat/                 # Periodic self-check service
  │   ├── events/                    # External system webhook handlers
  │   ├── memory/                    # Memory store, search, lifecycle
  │   ├── escalation/               # Escalation tracking + follow-up
  │   └── subagents/                # Sub-agent spawning + result collection
  │
  ├── identity/                      # Persona, rules, tools, profile
  │   ├── persona.py                 # AgentPersona CRUD
  │   ├── rules.py                   # AgentOperatingRules CRUD
  │   ├── tools.py                   # AgentToolConfig CRUD
  │   └── prompt_builder.py          # System prompt assembly
  │
  ├── observability/                 # Audit, metrics, dashboard
  │   ├── activity_log.py            # Activity logging
  │   ├── audit.py                   # Audit trail
  │   └── metrics.py                 # Dashboard metrics
  │
  └── guardrails/                    # Safety boundaries
      ├── action_limits.py           # Rate limits, approval gates
      └── content_policy.py          # Content guardrails
```
