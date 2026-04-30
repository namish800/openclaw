# OpenClaw System Prompt

This document describes the system prompt used by OpenClaw agents. The prompt is
not a single static string — it is dynamically assembled by
`buildAgentSystemPrompt` in `src/agents/system-prompt.ts:453`, with sections
included or omitted based on tools, channel, sandbox state, runtime info, and
caller flags.

## Prompt Modes

Set via `params.promptMode` (`src/agents/system-prompt.ts:476`).

- `"full"` (default, main agent): all sections.
- `"minimal"` (subagents): drops Authorized Senders, Memory, Self-Update, Model
  Aliases, Documentation, Assistant Output Directives, Webchat Embed, Messaging,
  Voice, Heartbeats, Execution Bias, Silent Replies, and Skills lookup
  guidance.
- `"none"` (`src/agents/system-prompt.ts:713-715`): collapses everything to a
  single line — `You are a personal assistant running inside OpenClaw.`

## Provider Overrides

`params.promptContribution` (`src/agents/system-prompt-contribution.ts`) lets
the active provider override or wrap content:

- `stablePrefix` — block inserted before `## Safety`.
- `dynamicSuffix` — block appended near the end (above `## Heartbeats`).
- `sectionOverrides` for `interaction_style`, `tool_call_style`, and
  `execution_bias`.

## Section Order (full mode, end to end)

1. Identity line: `You are a personal assistant running inside OpenClaw.`
2. `## Tooling` — filtered tool list, ACP guidance, sub-agent guidance.
3. `## Tool Call Style` (overridable).
4. `## Execution Bias` (overridable).
5. Provider stable prefix (if any).
6. `## Safety`.
7. `## OpenClaw CLI Quick Reference`.
8. `## Skills (mandatory)` — only if `skillsPrompt` is provided.
9. `## Memory` — `buildMemoryPromptSection` in `src/plugins/memory-state.ts`.
10. `## OpenClaw Self-Update` — only if the `gateway` tool is available.
11. `## Model Aliases` — only if `modelAliasLines` provided.
12. Time hint (one line).
13. `## Workspace` — workspace dir + sandbox-aware path guidance + workspace
    notes.
14. `## Documentation`.
15. `## Sandbox` — only when `sandboxInfo.enabled`.
16. `## Authorized Senders` — only with `ownerNumbers`.
17. `## Current Date & Time` — only with `userTimezone`.
18. `## Workspace Files (injected)` header.
19. `## Assistant Output Directives` — `MEDIA:`, `[[audio_as_voice]]`,
    `[[reply_to_current]]`, `[[reply_to:<id>]]`.
20. `## Reasoning Format` — only with `reasoningTagHint` (`<think>`/`<final>`).
21. `# Project Context` — stable workspace context files (see below).
22. `## Silent Replies` — uses `SILENT_REPLY_TOKEN`.
23. `<<< SYSTEM_PROMPT_CACHE_BOUNDARY >>>` — marker. Everything above is the
    cacheable prefix.
24. `# Dynamic Project Context` — dynamic context files below the cache
    boundary.
25. `## Control UI Embed` — only on `webchat`.
26. `## Messaging` — channel routing, `message` tool usage, inline buttons.
27. `## Voice (TTS)` — only with `ttsHint`.
28. `## Group Chat Context` (or `## Subagent Context` in minimal) — only with
    `extraSystemPrompt`.
29. `## Reactions` — minimal or extensive guidance.
30. Provider dynamic suffix (if any).
31. `## Heartbeats` — only with `heartbeatPrompt`.
32. `## Runtime` — single-line key=value summary + reasoning level line.

## Project Context Files

Callers pass `params.contextFiles: EmbeddedContextFile[]`. The builder splits
them around the cache boundary (`src/agents/system-prompt.ts:918-924`).

### Stable vs Dynamic Split

- **Stable** (above the cache boundary, under `# Project Context`): everything
  not listed below.
- **Dynamic** (below the cache boundary, under `# Dynamic Project Context`):
  basenames in `DYNAMIC_CONTEXT_FILE_BASENAMES`
  (`src/agents/system-prompt.ts:57`). Currently: `heartbeat.md`.

### HEARTBEAT.md gating

HEARTBEAT.md is *eligible* every session but the upstream
`resolveBootstrapFilesForRun` (`src/agents/bootstrap-files.ts:194-227`) will
strip it from `contextFiles` unless one of these holds:

- `runKind === "heartbeat"` (a heartbeat-triggered run), or
- the session agent is *not* the default agent, or
- `shouldIncludeHeartbeatGuidanceForSystemPrompt` returns true — i.e. the
  default agent has heartbeats enabled by agent policy, `heartbeat.every`
  parses to a positive cadence, and `heartbeat.includeSystemPromptSection` is
  not `false` (`src/agents/heartbeat-system-prompt.ts:54-72`).

Lightweight context mode (`applyContextModeFilter`,
`src/agents/bootstrap-files.ts:177-192`) further reduces heartbeat runs to
*only* HEARTBEAT.md, dropping AGENTS/SOUL/USER/etc.

The separate `## Heartbeats` system-prompt block (driven by `heartbeatPrompt`,
`src/agents/system-prompt.ts:130-141`) is gated even tighter for the embedded
runner: it is injected only when `trigger === "heartbeat"` and the agent is
the default agent (`shouldInjectHeartbeatPrompt` in
`src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts:213-231`,
`shouldInjectHeartbeatPromptForTrigger` in
`src/agents/pi-embedded-runner/run/trigger-policy.ts:11-22`). The CLI runner
path (`src/agents/cli-runner/prepare.ts:279`) does not gate by trigger and
includes the section whenever heartbeat policy/cadence are enabled.

### Stable File Ordering

`CONTEXT_FILE_ORDER` (`src/agents/system-prompt.ts:47-55`) defines a fixed sort
key by basename:

| Basename       | Order |
| -------------- | ----- |
| `agents.md`    | 10    |
| `soul.md`      | 20    |
| `identity.md`  | 30    |
| `user.md`      | 40    |
| `tools.md`     | 50    |
| `bootstrap.md` | 60    |
| `memory.md`    | 70    |

Files not in the table get `Number.MAX_SAFE_INTEGER` and sort alphabetically by
basename, then by full path (`sortContextFilesForPrompt`,
`src/agents/system-prompt.ts:80-96`).

### Rendered Shape

`buildProjectContextSection` (`src/agents/system-prompt.ts:98-128`) emits each
file as:

```
# Project Context

The following project context files have been loaded:
If SOUL.md is present, embody its persona and tone. Avoid stiff, generic replies; follow its guidance unless higher-priority instructions override it.

## <file.path>

<sanitized file.content>
```

The SOUL.md persona line is added only when a file with basename `soul.md` is
present in the stable set (`src/agents/system-prompt.ts:113-122`).

`sanitizeContextFileContentForPrompt` (`src/agents/system-prompt.ts:73-78`)
strips the default heartbeat policy quote and collapses runs of 3+ blank lines
to 2.

## Cache Boundary

`SYSTEM_PROMPT_CACHE_BOUNDARY` (`src/agents/system-prompt-cache-boundary.ts`)
is a marker emitted at `src/agents/system-prompt.ts:954`. Anthropic-family
transports use it to keep the large stable prompt prefix byte-identical across
turns and labs. Volatile content (dynamic context, channel guidance, group
chat context, runtime line) is intentionally placed below it so edits do not
invalidate the cached prefix.

## Runtime Line

Built by `buildRuntimeLine` (`src/agents/system-prompt.ts:1028`). Format:

```
Runtime: agent=<id> | host=<host> | repo=<repoRoot> | os=<os> (<arch>) | node=<node> | model=<model> | default_model=<defaultModel> | shell=<shell> | channel=<channel> | capabilities=<caps> | thinking=<level>
```

Followed by:

```
Reasoning: <level> (hidden unless on/stream). Toggle /reasoning; /status shows Reasoning when enabled.
```

## Source

- Builder: `src/agents/system-prompt.ts:453`
- Types: `src/agents/system-prompt.types.ts`
- Provider contribution: `src/agents/system-prompt-contribution.ts`
- Cache boundary marker: `src/agents/system-prompt-cache-boundary.ts`
- Memory section: `src/plugins/memory-state.ts`
- Bootstrap user-prompt prefix: `buildAgentUserPromptPrefix` at
  `src/agents/system-prompt.ts:192`
