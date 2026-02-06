# Coworker Gateway Layer — Design Document

## Design Principles (Extracted from OpenClaw)

This design follows seven architectural principles observed in the OpenClaw gateway codebase. Each principle is stated, then applied to the coworker platform.

### Principle 1: Stateless Plugins, External State

OpenClaw channel plugins hold no instance state. Every method receives its state as parameters — `cfg`, `account`, `accountId`, `abortSignal`, `getStatus()`, `setStatus()`. The `ChannelManager` owns all runtime state externally (abort controllers, running tasks, account snapshots).

**Applied here:** `ChannelPlugin` classes have no `self._state`. All per-account state is owned by the `ChannelManager`. Plugins receive a `ChannelContext` with everything they need and report state changes through it.

### Principle 2: Composable Optional Adapters, Not a Fixed Pipeline

OpenClaw's `ChannelPlugin` is not a base class with abstract methods in a fixed order. It's a struct of ~20 optional adapter slots (`security?`, `outbound?`, `threading?`, `mentions?`, `groups?`, etc.). Each channel implements only the adapters it needs. Shared infrastructure checks `if plugin.security:` before calling.

**Applied here:** `ChannelPlugin` is a `Protocol` with a required core (`config`, `gateway`, `outbound`) and optional adapter slots. No Template Method. The `MessagePipeline` composes adapter calls based on what the plugin provides.

### Principle 3: Two-Tier Weight (Dock vs Plugin)

OpenClaw splits channel metadata into a lightweight `ChannelDock` (capabilities, chunk limits, mention patterns — always loaded) and a heavy `ChannelPlugin` (full implementation with platform SDKs — loaded on demand). Shared code paths import docks, not plugins.

**Applied here:** `ChannelDescriptor` (light, always loaded) vs `ChannelPlugin` (heavy, loaded when starting accounts). Routing, allowlist checks, and UI rendering use descriptors. Agent invocation and delivery use the full plugin.

### Principle 4: Per-Account Lifecycle with Abort Signals

OpenClaw runs one `startAccount(ctx)` call per account, passing an `AbortController` for cancellation. The `ChannelManager` tracks per-account state in a `ChannelRuntimeStore` with three maps: `aborts` (cancellation), `tasks` (running promises), `runtimes` (snapshots). Starting an account that's already running is a no-op.

**Applied here:** `ChannelManager` tracks per-account lifecycle with `asyncio.Event` for cancellation. Each account gets its own task, snapshot, and cancel signal. Plugins receive a `ChannelGatewayContext` with the signal.

### Principle 5: Reply Dispatcher with Kind Discrimination

OpenClaw serializes all outbound messages through a `ReplyDispatcher` that classifies each piece as `"tool"`, `"block"`, or `"final"`. External channels (WhatsApp, Telegram) only receive finals. WebSocket clients receive everything (filtered by verbose level). Human-like delays are applied between blocks. The dispatcher is a serialized promise chain.

**Applied here:** `ReplyDispatcher` with `EventKind` discrimination. Streaming channels see all events. Buffered channels see only finals. The dispatcher serializes delivery and applies per-channel rate limiting.

### Principle 6: Config-Driven Multi-Tenancy

OpenClaw plugins receive `cfg: OpenClawConfig` and operate on it. They never read global state. This means multi-tenancy is achieved by passing a tenant-scoped config — the plugin code doesn't change.

**Applied here:** Plugins receive `TenantConfig` (loaded from DB, scoped to `org_id`). The plugin code is identical for all tenants. Tenant isolation happens at the config/context boundary, not inside plugin logic.

### Principle 7: Capabilities Declaration

OpenClaw channels declare `ChannelCapabilities` — which chat types they support, whether they handle polls, reactions, threads, media, native commands, block streaming. This feeds into agent prompts and tool availability.

**Applied here:** Every channel declares `ChannelCapabilities`. The agent receives capabilities in its context so it can adapt behavior (e.g., don't offer polls on email).

---

## Architecture Overview

```
         Channels
    (Web, Telegram*, Discord*, Email*)
              |
              v
     +------------------+
     |  MessagePipeline  |  Composed from plugin adapters (not a fixed pipeline)
     |                    |  normalize -> security -> resolve -> pre_invoke -> invoke -> dispatch
     +--------+----------+
              |
     +--------+----------+
     | ChannelManager     |  Owns all per-account state, lifecycle, health
     +--------+----------+
              |
     +--------+----------+
     | ReplyDispatcher    |  Serializes outbound: tool | block | final
     |                    |  Per-channel rate limits, coalescing, chunking
     +--------+----------+
              |
     +--------+----------+
     |  AgentInvoker      |  Protocol — abstracts agent from gateway
     |  (Protocol)        |  coworker_api provides concrete impl
     +--------+----------+
              |
     +--------+----------+
     | Agent (existing)   |  WorkflowOrchestrator + DeepAgents
     +--------+----------+
              |
         [agent events]
              |
     +--------+----------+
     | OutboundAdapter    |  Per-channel: chunk, format, deliver
     | (per channel)      |
     +--------+----------+
              |
         Channel Delivery
         (SSE, WS, Bot API, SMTP, etc.)
```

---

## Core Types

### Identity and Context

```python
@dataclass(frozen=True)
class ChannelIdentity:
    """Identifies a user on a specific platform. Immutable. Travels with every message."""
    channel_type: str              # "web", "telegram", "discord"
    account_id: str                # Which channel account (tenant-scoped)
    platform_user_id: str          # User ID on the platform
    platform_chat_id: str | None   # Group/channel/thread ID
    display_name: str | None
    metadata: dict[str, Any]       # Platform-specific extras (frozen via convention)


class ChatType(str, Enum):
    DIRECT = "direct"
    GROUP = "group"
    CHANNEL = "channel"
    THREAD = "thread"


class ContentType(str, Enum):
    TEXT = "text"
    IMAGE = "image"
    FILE = "file"
    AUDIO = "audio"
    VIDEO = "video"
    RICH = "rich"


@dataclass(frozen=True)
class ContentItem:
    type: ContentType
    text: str | None = None
    url: str | None = None
    mime_type: str | None = None
    metadata: dict[str, Any] = field(default_factory=dict)
```

### Normalized Message (Immutable After Creation)

Following OpenClaw's `MsgContext` — a rich, flat structure with all fields populated at creation time by the normalizer. No progressive mutation.

```python
@dataclass(frozen=True)
class NormalizedMessage:
    """Platform-agnostic message. Fully populated by the channel normalizer.

    OpenClaw equivalent: MsgContext (src/auto-reply/templating.ts:13-124).
    All fields set at creation. No progressive mutation."""

    # Identity
    id: str
    sender: ChannelIdentity
    chat_type: ChatType
    direction: str                      # "inbound" | "outbound"

    # Content
    body: str                           # Raw text body
    body_for_agent: str                 # Body with sender context for agent prompt
    body_for_commands: str              # Clean body for command parsing
    content: list[ContentItem]          # Structured content (media, files)

    # Routing (set by normalizer — NOT mutated later)
    org_id: str                         # Resolved during normalization
    session_id: str                     # Resolved during normalization
    thread_id: str | None               # LangGraph thread_id

    # Threading
    reply_to_id: str | None
    thread_label: str | None            # For thread-aware channels
    message_thread_id: str | None       # Platform thread ID

    # Sender metadata
    sender_name: str | None
    sender_id: str | None
    sender_username: str | None

    # Context
    timestamp: datetime
    was_mentioned: bool                 # Was the bot @mentioned?
    originating_channel: str | None     # For cross-channel reply routing
    originating_to: str | None
    metadata: dict[str, Any]            # Mode, mode_config, platform extras
```

### Channel Capabilities

```python
@dataclass(frozen=True)
class ChannelCapabilities:
    """Declares what this channel supports. Feeds into agent context.

    OpenClaw equivalent: ChannelCapabilities (src/channels/plugins/types.core.ts:164-177)."""

    chat_types: tuple[ChatType, ...]    # Which chat types this channel handles
    media: bool = False                 # Can send/receive images, files, audio
    polls: bool = False
    reactions: bool = False
    threads: bool = False
    edit: bool = False                  # Can edit sent messages
    unsend: bool = False                # Can delete sent messages
    rich_formatting: bool = False       # Buttons, cards, embeds
    native_commands: bool = False       # /slash commands
    block_streaming: bool = False       # Can receive incremental text updates
    max_text_length: int = 4096
```

### Account Configuration and State

```python
@dataclass
class ChannelAccountConfig:
    """Configuration for a channel account. Stored in DB.

    OpenClaw equivalent: resolved account from ChannelConfigAdapter."""
    id: str
    org_id: str
    channel_type: str
    name: str
    enabled: bool
    config: dict[str, Any]              # Channel-specific settings
    credentials: dict[str, Any]         # Encrypted credentials


class AccountState(str, Enum):
    CONFIGURED = "configured"
    NOT_CONFIGURED = "not_configured"
    ENABLED = "enabled"
    DISABLED = "disabled"
    LINKED = "linked"
    NOT_LINKED = "not_linked"


@dataclass
class ChannelAccountSnapshot:
    """Runtime state for a single channel account. Owned by ChannelManager, NOT the plugin.

    OpenClaw equivalent: ChannelAccountSnapshot (src/channels/plugins/types.core.ts:95-142)."""

    account_id: str
    name: str | None = None
    enabled: bool = False
    configured: bool = False
    running: bool = False
    connected: bool = False
    reconnect_attempts: int = 0
    last_connected_at: datetime | None = None
    last_disconnect_at: datetime | None = None
    last_disconnect_error: str | None = None
    last_message_at: datetime | None = None
    last_inbound_at: datetime | None = None
    last_outbound_at: datetime | None = None
    last_error: str | None = None
    last_start_at: datetime | None = None
    last_stop_at: datetime | None = None
    dm_policy: str | None = None
    allow_from: list[str] | None = None
    probe: Any = None                   # Channel-specific health probe result
    audit: Any = None                   # Channel-specific audit result
```

### Reply Payload

```python
@dataclass(frozen=True)
class ReplyPayload:
    """What the agent produces. Channel-agnostic.

    OpenClaw equivalent: ReplyPayload (src/auto-reply/types.ts:44-57)."""

    text: str | None = None
    media_url: str | None = None
    media_urls: list[str] | None = None
    reply_to_id: str | None = None
    audio_as_voice: bool = False
    is_error: bool = False
    channel_data: dict[str, Any] | None = None  # Per-channel extras (buttons, cards)


class EventKind(str, Enum):
    """Discriminator for outbound message types.

    OpenClaw equivalent: ReplyDispatchKind (src/auto-reply/reply/reply-dispatcher.ts:8)."""

    TOOL = "tool"       # Tool call results — internal channels only
    BLOCK = "block"     # Streaming text block — streaming channels only
    FINAL = "final"     # Final reply — ALL channels receive this
```

---

## Plugin Interface (Composable Adapters)

### ChannelDescriptor (Lightweight — Always Loaded)

```python
@dataclass(frozen=True)
class ChannelDescriptor:
    """Lightweight channel metadata for shared code paths. No heavy deps.

    OpenClaw equivalent: ChannelDock (src/channels/dock.ts:40-64).
    Used by routing, allowlist checks, UI rendering. Never loads platform SDKs."""

    id: str
    label: str
    docs_path: str
    blurb: str
    capabilities: ChannelCapabilities
    aliases: tuple[str, ...] = ()
    order: int = 999

    # Lightweight behavioral hints (no heavy imports)
    text_chunk_limit: int = 4096
    debounce_ms: int = 0                    # Inbound message debounce
    block_streaming_coalesce_min_chars: int | None = None
    block_streaming_coalesce_idle_ms: int | None = None
    enforce_owner_for_commands: bool = False
```

### ChannelPlugin (Full Implementation — Loaded on Demand)

```python
class ChannelPlugin(Protocol):
    """Main extension point. Composable adapters — implement what you need.

    OpenClaw equivalent: ChannelPlugin<ResolvedAccount> (src/channels/plugins/types.plugin.ts:48-84).

    Required: descriptor, config, outbound
    Optional: all other adapters. Infrastructure checks presence before calling.

    Plugins are STATELESS. All per-account state is owned by ChannelManager.
    Methods receive context objects with everything they need."""

    # ── Required ────────────────────────────────────────────────

    descriptor: ChannelDescriptor

    config: "ConfigAdapter"

    # ── Optional Adapters ───────────────────────────────────────
    # OpenClaw makes EVERYTHING except id/meta/capabilities/config optional.
    # Infrastructure checks `if plugin.outbound:` before calling.

    outbound: "OutboundAdapter | None"           # Platform delivery (most channels need this)
    gateway: "GatewayAdapter | None"            # Lifecycle (start/stop accounts)
    security: "SecurityAdapter | None"          # DM policy, allowlist
    pairing: "PairingAdapter | None"            # Allowlist management UI
    status: "StatusAdapter | None"              # Health probes, auditing
    mentions: "MentionAdapter | None"           # Strip @mention patterns
    groups: "GroupAdapter | None"               # Group mention policy, tool policy
    threading: "ThreadingAdapter | None"        # Reply-to mode, thread context
    messaging: "MessagingAdapter | None"        # Target normalization
    directory: "DirectoryAdapter | None"        # List peers, groups, members
    commands: "CommandAdapter | None"           # In-channel commands (/start, /help)
    actions: "MessageActionAdapter | None"      # Reactions, edits, unsend
    auth: "AuthAdapter | None"                  # Login flows (QR, OAuth)
    onboarding: "OnboardingAdapter | None"      # Setup wizard hooks
    agent_prompt: "AgentPromptAdapter | None"   # Hints for the AI agent
    elevated: "ElevatedAdapter | None"          # Elevated/admin operations
    setup: "SetupAdapter | None"               # Post-config setup tasks
    heartbeat: "HeartbeatAdapter | None"        # Keep-alive/heartbeat handling
    resolver: "ResolverAdapter | None"          # Resolve targets/identifiers
    streaming: "StreamingAdapter | None"        # Block streaming support
```

### Adapter Protocols

Each adapter is a separate protocol. Channels implement only what they need.

```python
# ── Config Adapter (REQUIRED) ─────────────────────────────────
# OpenClaw equivalent: ChannelConfigAdapter (types.adapters.ts:41-65)

class ConfigAdapter(Protocol):
    """Maps between stored config and typed account objects."""

    def list_account_ids(self, org_id: str) -> list[str]: ...

    def resolve_account(
        self, account_config: ChannelAccountConfig
    ) -> Any:
        """Hydrate DB config into a typed account object."""
        ...

    def is_enabled(self, account: Any) -> bool: ...

    def is_configured(self, account: Any) -> bool: ...

    def describe_account(
        self, account: Any
    ) -> ChannelAccountSnapshot: ...

    async def normalize_raw(
        self, raw_message: Any, account_config: ChannelAccountConfig
    ) -> "NormalizedMessage":
        """Channel-specific normalization of raw platform input into NormalizedMessage.
        Each channel implements this to convert its native format (Telegram Update,
        Discord Message, etc.) into the shared NormalizedMessage type.
        org_id and session_id may be left empty — the pipeline fills them."""
        ...


# ── Gateway Adapter (lifecycle) ────────────────────────────────
# OpenClaw equivalent: ChannelGatewayAdapter (types.adapters.ts:194-208)

@dataclass
class ChannelGatewayContext:
    """Context passed to start_account/stop_account. Plugin reads from this, doesn't own it.

    OpenClaw equivalent: ChannelGatewayContext (types.adapters.ts:149-158)."""
    account_config: ChannelAccountConfig
    account: Any                        # Typed account from ConfigAdapter.resolve_account
    account_id: str
    org_id: str
    cancel_event: asyncio.Event         # Set when account should stop
    get_status: Callable[[], ChannelAccountSnapshot]
    set_status: Callable[[ChannelAccountSnapshot], None]
    log: "Logger"


class GatewayAdapter(Protocol):
    """Lifecycle management for channel accounts."""

    async def start_account(self, ctx: ChannelGatewayContext) -> None:
        """Boot the channel connection. Runs until cancel_event is set or error.
        Call ctx.set_status() to report connected/disconnected/error."""
        ...

    async def stop_account(self, ctx: ChannelGatewayContext) -> None:
        """Graceful teardown. Called before cancel_event is set."""
        ...

    async def login_with_qr_start(
        self, account_id: str, timeout_ms: int = 60000
    ) -> dict[str, Any]:
        """Begin QR auth flow. Returns {"qr_data_url": "...", "message": "..."}."""
        ...

    async def login_with_qr_wait(
        self, account_id: str, timeout_ms: int = 120000
    ) -> dict[str, Any]:
        """Wait for QR scan completion. Returns {"connected": bool, "message": "..."}."""
        ...

    async def logout_account(self, ctx: ChannelGatewayContext) -> dict[str, Any]:
        """Clear credentials, disconnect. Returns {"cleared": bool}."""
        ...


# ── Outbound Adapter (REQUIRED) ───────────────────────────────
# OpenClaw equivalent: ChannelOutboundAdapter (types.adapters.ts:89-106)

class DeliveryMode(str, Enum):
    DIRECT = "direct"           # Plugin sends directly to platform
    GATEWAY = "gateway"         # Route through gateway RPC
    HYBRID = "hybrid"           # Some direct, some gateway


class OutboundAdapter(Protocol):
    """Delivers messages to the platform."""

    delivery_mode: DeliveryMode
    text_chunk_limit: int
    chunker_mode: str                   # "text" | "markdown"

    async def send_text(
        self,
        to: str,
        text: str,
        *,
        account_id: str | None = None,
        reply_to_id: str | None = None,
        thread_id: str | None = None,
    ) -> "OutboundDeliveryResult": ...

    async def send_media(
        self,
        to: str,
        text: str | None,
        media_url: str,
        *,
        account_id: str | None = None,
        reply_to_id: str | None = None,
    ) -> "OutboundDeliveryResult": ...

    async def send_payload(
        self,
        to: str,
        payload: ReplyPayload,
        *,
        account_id: str | None = None,
    ) -> "OutboundDeliveryResult": ...


@dataclass
class OutboundDeliveryResult:
    channel: str
    message_id: str
    chat_id: str | None = None
    timestamp: float | None = None


# ── Security Adapter ───────────────────────────────────────────
# OpenClaw equivalent: ChannelSecurityAdapter (types.adapters.ts:307-312)

@dataclass(frozen=True)
class DmPolicy:
    """Resolved DM policy for an account."""
    policy: str                         # "owner" | "allowlist" | "everyone"
    allow_from: list[str] | None
    approve_hint: str                   # CLI/UI hint for approving users
    normalize_entry: Callable[[str], str] | None = None


class SecurityAdapter(Protocol):
    """DM policy resolution and security warnings."""

    def resolve_dm_policy(
        self, account: Any, account_config: ChannelAccountConfig
    ) -> DmPolicy | None: ...

    async def collect_warnings(
        self, account: Any, account_config: ChannelAccountConfig
    ) -> list[str]: ...


# ── Mention Adapter ────────────────────────────────────────────
# OpenClaw equivalent: ChannelMentionAdapter (types.core.ts:194-206)

class MentionAdapter(Protocol):
    """Strip @mention patterns from message body before agent sees it."""

    def strip_patterns(
        self, message: NormalizedMessage, account_config: ChannelAccountConfig
    ) -> list[str]:
        """Return regex patterns to strip (e.g., ['<@!?\\d+>'] for Discord)."""
        ...

    def strip_mentions(
        self, text: str, message: NormalizedMessage, account_config: ChannelAccountConfig
    ) -> str:
        """Return text with mentions removed."""
        ...


# ── Group Adapter ──────────────────────────────────────────────
# OpenClaw equivalent: ChannelGroupAdapter (types.adapters.ts:67-71)

class GroupAdapter(Protocol):
    """Group-specific behavior: mention requirements, tool policies."""

    def resolve_require_mention(
        self, *, group_id: str | None, account_config: ChannelAccountConfig
    ) -> bool | None:
        """Whether the bot requires @mention in this group. None = use default."""
        ...

    def resolve_tool_policy(
        self, *, group_id: str | None, account_config: ChannelAccountConfig
    ) -> dict[str, Any] | None:
        """Tool restrictions for this group. None = no restrictions."""
        ...


# ── Threading Adapter ──────────────────────────────────────────
# OpenClaw equivalent: ChannelThreadingAdapter (types.core.ts:215-228)

class ThreadingAdapter(Protocol):
    """Reply-to behavior varies by channel."""

    def resolve_reply_to_mode(
        self, *, account_config: ChannelAccountConfig, chat_type: str | None
    ) -> str:
        """Return "off" | "first" | "all"."""
        ...

    def build_tool_context(
        self, *, message: NormalizedMessage, account_config: ChannelAccountConfig
    ) -> dict[str, Any] | None:
        """Build threading context for agent tools (e.g., Slack thread_ts)."""
        ...


# ── Directory Adapter ──────────────────────────────────────────
# OpenClaw equivalent: ChannelDirectoryAdapter (types.adapters.ts:232-273)

@dataclass
class DirectoryEntry:
    kind: str                           # "user" | "group" | "channel"
    id: str
    name: str | None = None
    handle: str | None = None
    avatar_url: str | None = None


class DirectoryAdapter(Protocol):
    """Contact and group discovery."""

    async def list_peers(
        self, *, account_config: ChannelAccountConfig, query: str | None = None, limit: int | None = None
    ) -> list[DirectoryEntry]: ...

    async def list_groups(
        self, *, account_config: ChannelAccountConfig, query: str | None = None, limit: int | None = None
    ) -> list[DirectoryEntry]: ...

    async def list_group_members(
        self, *, account_config: ChannelAccountConfig, group_id: str, limit: int | None = None
    ) -> list[DirectoryEntry]: ...


# ── Command Adapter ────────────────────────────────────────────
# OpenClaw equivalent: ChannelCommandAdapter (types.core.ts:302-305)
# + command handling in auto-reply pipeline

class CommandAdapter(Protocol):
    """Handle in-channel commands (/start, /help, /stop, etc.)."""

    def recognize_command(self, message: NormalizedMessage) -> str | None:
        """Return command name if this message is a command, None otherwise."""
        ...

    async def handle_command(
        self, command: str, message: NormalizedMessage, *, account_config: ChannelAccountConfig
    ) -> ReplyPayload | None:
        """Handle the command. Return a reply payload or None to continue to agent."""
        ...

    enforce_owner_for_commands: bool


# ── Status Adapter ─────────────────────────────────────────────
# OpenClaw equivalent: ChannelStatusAdapter (types.adapters.ts:108-147)

class StatusAdapter(Protocol):
    """Health probes and auditing."""

    async def probe_account(
        self, *, account: Any, account_config: ChannelAccountConfig, timeout_ms: int
    ) -> Any:
        """Active health check (ping API, verify webhook)."""
        ...

    async def audit_account(
        self, *, account: Any, account_config: ChannelAccountConfig, timeout_ms: int, probe: Any = None
    ) -> Any:
        """Deeper audit (check permissions, scopes, webhook config)."""
        ...

    def build_account_snapshot(
        self, *, account: Any, account_config: ChannelAccountConfig, probe: Any = None, audit: Any = None
    ) -> ChannelAccountSnapshot: ...

    def collect_status_issues(
        self, accounts: list[ChannelAccountSnapshot]
    ) -> list[dict[str, Any]]: ...


# ── Message Action Adapter ─────────────────────────────────────
# OpenClaw equivalent: ChannelMessageActionAdapter (types.core.ts:309-316)

class MessageActionAdapter(Protocol):
    """Reactions, edits, unsend."""

    def supported_actions(self) -> list[str]:
        """e.g., ["react", "edit", "unsend", "pin"]"""
        ...

    async def handle_action(
        self, action: str, params: dict[str, Any], *, account_config: ChannelAccountConfig
    ) -> dict[str, Any]: ...


# ── Pairing Adapter ────────────────────────────────────────────
# OpenClaw equivalent: ChannelPairingAdapter (types.adapters.ts:184-192)

class PairingAdapter(Protocol):
    """Allowlist management for approving new users."""

    id_label: str                       # "Telegram user ID", "Discord user ID"

    def normalize_allow_entry(self, entry: str) -> str: ...

    async def notify_approval(
        self, *, user_id: str, account_config: ChannelAccountConfig
    ) -> None: ...


# ── Agent Prompt Adapter ───────────────────────────────────────
# OpenClaw equivalent: ChannelAgentPromptAdapter (types.core.ts:268-270)

class AgentPromptAdapter(Protocol):
    """Channel-specific hints for the AI agent."""

    def message_tool_hints(self, *, account_config: ChannelAccountConfig) -> list[str]:
        """Return hints to include in the agent system prompt for this channel."""
        ...


# ── Auth Adapter ───────────────────────────────────────────────
# OpenClaw equivalent: ChannelAuthAdapter (types.adapters.ts:210-218)

class AuthAdapter(Protocol):
    """Channel-specific login flows (QR, OAuth, token, etc.)."""

    async def login(
        self, *, account_config: ChannelAccountConfig, channel_input: str | None = None
    ) -> None: ...


# ── Onboarding Adapter ────────────────────────────────────────
# OpenClaw equivalent: ChannelOnboardingAdapter (onboarding-types.ts:79-86)

class OnboardingAdapter(Protocol):
    """Setup wizard hooks for channel configuration UI."""

    async def get_status(self, *, account_config: ChannelAccountConfig) -> dict[str, Any]: ...

    async def configure(
        self, *, account_config: ChannelAccountConfig, input: dict[str, Any]
    ) -> ChannelAccountConfig: ...
```

---

## ChannelManager — External Lifecycle and State

```python
@dataclass
class _AccountRuntime:
    """Per-account runtime state. Owned by ChannelManager, NOT the plugin.

    OpenClaw equivalent: ChannelRuntimeStore (src/gateway/server-channels.ts:18-22)."""
    cancel_event: asyncio.Event
    task: asyncio.Task | None
    snapshot: ChannelAccountSnapshot


class ChannelManager:
    """Manages channel plugin lifecycle and per-account state.

    OpenClaw equivalent: createChannelManager (src/gateway/server-channels.ts:64-308).

    Key invariant: plugins are STATELESS. All mutable state lives here.
    The plugin receives a ChannelGatewayContext with getters/setters that
    read/write the snapshot owned by this manager."""

    def __init__(
        self,
        registry: "ChannelRegistry",
        account_repo: "ChannelAccountRepo",
    ):
        self._registry = registry
        self._account_repo = account_repo
        # (channel_type, account_id) -> runtime
        self._runtimes: dict[tuple[str, str], _AccountRuntime] = {}

    def get_snapshot(self, channel_type: str, account_id: str) -> ChannelAccountSnapshot:
        key = (channel_type, account_id)
        runtime = self._runtimes.get(key)
        if runtime:
            return runtime.snapshot
        return ChannelAccountSnapshot(account_id=account_id)

    def _set_snapshot(self, channel_type: str, account_id: str, snapshot: ChannelAccountSnapshot) -> None:
        key = (channel_type, account_id)
        runtime = self._runtimes.get(key)
        if runtime:
            runtime.snapshot = snapshot

    async def start_account(self, channel_type: str, account_id: str) -> None:
        """Start a single channel account.

        OpenClaw equivalent: startChannel (server-channels.ts:96-178)."""
        key = (channel_type, account_id)
        if key in self._runtimes and self._runtimes[key].task is not None:
            return  # Already running — no-op

        plugin = self._registry.get(channel_type)
        if not plugin or not plugin.gateway:
            return

        account_config = await self._account_repo.get_by_id(account_id)
        if not account_config:
            return

        account = plugin.config.resolve_account(account_config)
        if not plugin.config.is_enabled(account):
            self._runtimes[key] = _AccountRuntime(
                cancel_event=asyncio.Event(),
                task=None,
                snapshot=ChannelAccountSnapshot(
                    account_id=account_id, running=False, last_error="disabled"
                ),
            )
            return

        if not plugin.config.is_configured(account):
            self._runtimes[key] = _AccountRuntime(
                cancel_event=asyncio.Event(),
                task=None,
                snapshot=ChannelAccountSnapshot(
                    account_id=account_id, running=False, last_error="not configured"
                ),
            )
            return

        cancel_event = asyncio.Event()
        snapshot = ChannelAccountSnapshot(
            account_id=account_id,
            running=True,
            last_start_at=datetime.utcnow(),
            last_error=None,
        )
        runtime = _AccountRuntime(cancel_event=cancel_event, task=None, snapshot=snapshot)
        self._runtimes[key] = runtime

        ctx = ChannelGatewayContext(
            account_config=account_config,
            account=account,
            account_id=account_id,
            org_id=account_config.org_id,
            cancel_event=cancel_event,
            get_status=lambda: self.get_snapshot(channel_type, account_id),
            set_status=lambda s: self._set_snapshot(channel_type, account_id, s),
            log=logging.getLogger(f"gateway.channels.{channel_type}.{account_id}"),
        )

        async def _run():
            try:
                await plugin.gateway.start_account(ctx)
            except Exception as e:
                self._set_snapshot(channel_type, account_id, ChannelAccountSnapshot(
                    account_id=account_id, last_error=str(e),
                ))
            finally:
                runtime.task = None
                self._set_snapshot(channel_type, account_id, ChannelAccountSnapshot(
                    account_id=account_id, running=False, last_stop_at=datetime.utcnow(),
                ))

        runtime.task = asyncio.create_task(_run())

    async def stop_account(self, channel_type: str, account_id: str) -> None:
        """Stop a single channel account.

        OpenClaw equivalent: stopChannel (server-channels.ts:181-230)."""
        key = (channel_type, account_id)
        runtime = self._runtimes.get(key)
        if not runtime:
            return

        plugin = self._registry.get(channel_type)

        # Signal cancellation
        runtime.cancel_event.set()

        # Call plugin's graceful stop if available
        if plugin and plugin.gateway:
            try:
                account_config = await self._account_repo.get_by_id(account_id)
                if account_config:
                    account = plugin.config.resolve_account(account_config)
                    ctx = ChannelGatewayContext(
                        account_config=account_config,
                        account=account,
                        account_id=account_id,
                        org_id=account_config.org_id,
                        cancel_event=runtime.cancel_event,
                        get_status=lambda: self.get_snapshot(channel_type, account_id),
                        set_status=lambda s: self._set_snapshot(channel_type, account_id, s),
                        log=logging.getLogger(f"gateway.channels.{channel_type}.{account_id}"),
                    )
                    await plugin.gateway.stop_account(ctx)
            except Exception:
                pass

        # Wait for task to finish
        if runtime.task:
            try:
                await asyncio.wait_for(runtime.task, timeout=10.0)
            except (asyncio.TimeoutError, Exception):
                runtime.task.cancel()

        self._set_snapshot(channel_type, account_id, ChannelAccountSnapshot(
            account_id=account_id, running=False, last_stop_at=datetime.utcnow(),
        ))

    async def start_all(self) -> None:
        """Start all active channel accounts across all tenants."""
        accounts = await self._account_repo.list_active()
        for account in accounts:
            await self.start_account(account.channel_type, account.id)

    async def stop_all(self) -> None:
        """Stop all running accounts."""
        for (channel_type, account_id) in list(self._runtimes.keys()):
            await self.stop_account(channel_type, account_id)

    def get_runtime_snapshot(self) -> dict[str, dict[str, ChannelAccountSnapshot]]:
        """Return all account snapshots grouped by channel type.

        OpenClaw equivalent: getRuntimeSnapshot (server-channels.ts:262-298)."""
        result: dict[str, dict[str, ChannelAccountSnapshot]] = {}
        for (channel_type, account_id), runtime in self._runtimes.items():
            if channel_type not in result:
                result[channel_type] = {}
            result[channel_type][account_id] = runtime.snapshot
        return result
```

---

## Reply Dispatcher

```python
class ReplyDispatcher:
    """Serializes outbound replies with kind discrimination.

    OpenClaw equivalent: createReplyDispatcher (src/auto-reply/reply/reply-dispatcher.ts:99-162).

    Key behaviors:
    - All replies serialized through an async queue (preserves tool/block/final order)
    - Human-like delays between block replies (configurable)
    - Normalize and filter before delivery (strip silent tokens, empty payloads)
    - Deliver callback receives (payload, kind) — channels decide what to do with each kind
    """

    def __init__(
        self,
        deliver: Callable[[ReplyPayload, EventKind], Awaitable[None]],
        on_skip: Callable[[ReplyPayload, EventKind, str], None] | None = None,
        human_delay_min_ms: int = 0,
        human_delay_max_ms: int = 0,
    ):
        self._deliver = deliver
        self._on_skip = on_skip
        self._human_delay_min_ms = human_delay_min_ms
        self._human_delay_max_ms = human_delay_max_ms
        self._queue: asyncio.Queue[tuple[ReplyPayload, EventKind] | None] = asyncio.Queue()
        self._counts: dict[str, int] = {"tool": 0, "block": 0, "final": 0}
        self._sent_first_block = False
        self._task = asyncio.create_task(self._process())

    async def _process(self):
        while True:
            item = await self._queue.get()
            if item is None:
                break
            payload, kind = item
            try:
                # Human delay between block replies (not first, not tool/final)
                if kind == EventKind.BLOCK and self._sent_first_block:
                    delay = self._human_delay()
                    if delay > 0:
                        await asyncio.sleep(delay / 1000)
                if kind == EventKind.BLOCK:
                    self._sent_first_block = True

                await self._deliver(payload, kind)
            except Exception:
                pass  # on_error could be added
            finally:
                self._queue.task_done()

    def _human_delay(self) -> int:
        if self._human_delay_max_ms <= self._human_delay_min_ms:
            return self._human_delay_min_ms
        import random
        return random.randint(self._human_delay_min_ms, self._human_delay_max_ms)

    def send_tool_result(self, payload: ReplyPayload) -> bool:
        return self._enqueue(payload, EventKind.TOOL)

    def send_block_reply(self, payload: ReplyPayload) -> bool:
        return self._enqueue(payload, EventKind.BLOCK)

    def send_final_reply(self, payload: ReplyPayload) -> bool:
        return self._enqueue(payload, EventKind.FINAL)

    def _enqueue(self, payload: ReplyPayload, kind: EventKind) -> bool:
        normalized = self._normalize(payload)
        if normalized is None:
            if self._on_skip:
                self._on_skip(payload, kind, "empty_or_silent")
            return False
        self._counts[kind.value] += 1
        self._queue.put_nowait((normalized, kind))
        return True

    def _normalize(self, payload: ReplyPayload) -> ReplyPayload | None:
        """Filter empty/silent/heartbeat payloads and sanitize text.

        OpenClaw equivalent: normalizeReplyPayload (normalize-reply.ts:23-94).

        Full normalization pipeline (matching OpenClaw):
        1. Empty check — no text, no media, no channel_data → skip ("empty")
        2. Silent token — text matches SILENT_REPLY_TOKEN → skip ("silent")
        3. Heartbeat token — strip HEARTBEAT_TOKEN from text → skip if nothing remains ("heartbeat")
        4. Sanitize — strip embedded PI/agent control sequences from user-facing text
        5. Response prefix — prepend configured prefix (e.g., agent name)
        """
        has_media = bool(payload.media_url or payload.media_urls)
        has_channel_data = bool(payload.channel_data)
        text = (payload.text or "").strip()

        # 1. Empty check
        if not text and not has_media and not has_channel_data:
            return None

        # 2. Silent token check
        if text and self._is_silent(text):
            if not has_media and not has_channel_data:
                return None
            text = ""

        # 3. Heartbeat token strip
        if text and self._HEARTBEAT_TOKEN in text:
            text = text.replace(self._HEARTBEAT_TOKEN, "").strip()
            if not text and not has_media and not has_channel_data:
                return None

        # 4. Sanitize user-facing text
        if text:
            text = self._sanitize(text)

        if not text and not has_media and not has_channel_data:
            return None

        return ReplyPayload(
            text=text or None,
            media_url=payload.media_url,
            media_urls=payload.media_urls,
            reply_to_id=payload.reply_to_id,
            audio_as_voice=payload.audio_as_voice,
            is_error=payload.is_error,
            channel_data=payload.channel_data,
        )

    _SILENT_TOKEN = "[silent]"
    _HEARTBEAT_TOKEN = "[heartbeat]"

    def _is_silent(self, text: str) -> bool:
        return text.strip().lower() == self._SILENT_TOKEN

    def _sanitize(self, text: str) -> str:
        """Strip embedded control sequences not meant for end users."""
        # Implementation: remove agent-internal markers, control chars, etc.
        return text

    async def wait_for_idle(self) -> None:
        await self._queue.join()

    def get_queued_counts(self) -> dict[str, int]:
        return dict(self._counts)

    async def close(self) -> None:
        self._queue.put_nowait(None)
        await self._task
```

---

## Block Streaming Coalescer

```python
class BlockStreamingCoalescer:
    """Buffers block replies and flushes when thresholds are met.

    OpenClaw equivalent: block-streaming.ts (src/auto-reply/reply/block-streaming.ts).

    Sits between the agent event stream and the ReplyDispatcher for block events.
    Accumulates text until one of these conditions triggers a flush:
    - Text exceeds min_chars AND a natural break point is found (paragraph/sentence)
    - Text exceeds max_chars (hard limit — flush regardless)
    - idle_ms elapses since last text arrived (idle timeout)

    Defaults (from OpenClaw):
    - min_chars: 800
    - max_chars: 1200
    - idle_ms: 1000
    - break_preference: "paragraph" (also supports "newline", "sentence")
    """

    DEFAULT_MIN_CHARS = 800
    DEFAULT_MAX_CHARS = 1200
    DEFAULT_IDLE_MS = 1000

    def __init__(
        self,
        on_flush: Callable[[str], Awaitable[None]],
        min_chars: int = DEFAULT_MIN_CHARS,
        max_chars: int = DEFAULT_MAX_CHARS,
        idle_ms: int = DEFAULT_IDLE_MS,
        break_preference: str = "paragraph",
    ):
        self._on_flush = on_flush
        self._min_chars = min_chars
        self._max_chars = max_chars
        self._idle_ms = idle_ms
        self._break_preference = break_preference
        self._buffer = ""
        self._idle_timer: asyncio.TimerHandle | None = None

    async def append(self, text: str) -> None:
        """Append text to the buffer and flush if thresholds are met."""
        self._cancel_idle_timer()
        self._buffer += text

        # Hard limit: flush immediately
        if len(self._buffer) >= self._max_chars:
            await self._flush_at_break(self._max_chars)
            return

        # Soft limit: flush at natural break if we have enough text
        if len(self._buffer) >= self._min_chars:
            break_pos = self._find_break(self._buffer, self._min_chars)
            if break_pos is not None:
                await self._flush_up_to(break_pos)
                return

        # Start idle timer
        self._start_idle_timer()

    async def flush_remaining(self) -> None:
        """Flush any remaining buffered text (called at end of stream)."""
        self._cancel_idle_timer()
        if self._buffer:
            text = self._buffer
            self._buffer = ""
            await self._on_flush(text)

    async def _flush_at_break(self, max_pos: int) -> None:
        break_pos = self._find_break(self._buffer, max_pos) or max_pos
        await self._flush_up_to(break_pos)

    async def _flush_up_to(self, pos: int) -> None:
        text = self._buffer[:pos]
        self._buffer = self._buffer[pos:]
        if text.strip():
            await self._on_flush(text)

    def _find_break(self, text: str, within: int) -> int | None:
        """Find a natural break point within the first `within` chars."""
        chunk = text[:within]
        if self._break_preference == "paragraph":
            idx = chunk.rfind("\n\n")
            if idx >= 0:
                return idx + 2
        if self._break_preference in ("paragraph", "newline"):
            idx = chunk.rfind("\n")
            if idx >= 0:
                return idx + 1
        if self._break_preference == "sentence":
            for sep in (". ", "! ", "? "):
                idx = chunk.rfind(sep)
                if idx >= 0:
                    return idx + len(sep)
        return None

    def _start_idle_timer(self) -> None:
        loop = asyncio.get_event_loop()
        self._idle_timer = loop.call_later(
            self._idle_ms / 1000, lambda: asyncio.ensure_future(self.flush_remaining())
        )

    def _cancel_idle_timer(self) -> None:
        if self._idle_timer:
            self._idle_timer.cancel()
            self._idle_timer = None
```

### Outbound Text Chunking

```python
class TextChunker:
    """Splits long outbound text into platform-safe chunks.

    OpenClaw equivalent: chunk.ts (src/auto-reply/chunk.ts).

    Applied AFTER coalescing, BEFORE platform delivery. Each channel's
    text_chunk_limit (from ChannelDescriptor) determines the max size.

    Modes:
    - "length": split at max length, preferring line/word boundaries
    - "markdown": markdown-aware splitting (preserve code blocks, lists, headers)
    - "newline": split on paragraph boundaries (double newline)
    """

    @staticmethod
    def chunk(text: str, limit: int, mode: str = "length") -> list[str]:
        if len(text) <= limit:
            return [text]
        if mode == "newline":
            return TextChunker._chunk_by_paragraphs(text, limit)
        if mode == "markdown":
            return TextChunker._chunk_markdown_aware(text, limit)
        return TextChunker._chunk_by_length(text, limit)

    @staticmethod
    def _chunk_by_length(text: str, limit: int) -> list[str]:
        chunks = []
        while len(text) > limit:
            # Prefer line boundary
            idx = text.rfind("\n", 0, limit)
            if idx < limit // 2:
                # Prefer word boundary
                idx = text.rfind(" ", 0, limit)
            if idx < limit // 4:
                idx = limit  # Hard split
            chunks.append(text[:idx].rstrip())
            text = text[idx:].lstrip()
        if text.strip():
            chunks.append(text)
        return chunks

    @staticmethod
    def _chunk_by_paragraphs(text: str, limit: int) -> list[str]:
        paragraphs = text.split("\n\n")
        chunks, current = [], ""
        for para in paragraphs:
            candidate = (current + "\n\n" + para).strip() if current else para
            if len(candidate) > limit and current:
                chunks.append(current)
                current = para
            else:
                current = candidate
        if current:
            chunks.append(current)
        return chunks

    @staticmethod
    def _chunk_markdown_aware(text: str, limit: int) -> list[str]:
        """Split while preserving markdown code blocks and structure."""
        # Implementation: track code fence state, prefer splitting outside fences
        return TextChunker._chunk_by_length(text, limit)  # Fallback for now
```

---

## Message Pipeline (Composable, Not Fixed)

```python
class MessagePipeline:
    """Composes adapter calls based on what the plugin provides.

    NOT a Template Method. Instead, each stage checks if the relevant adapter
    exists and calls it. Stages can be skipped, reordered, or extended.

    OpenClaw equivalent: the combination of:
    - process-message.ts (per-channel monitor)
    - dispatch.ts (dispatchInboundMessage)
    - dispatch-from-config.ts (dispatchReplyFromConfig)
    - inbound-context.ts (finalizeInboundContext)
    """

    def __init__(
        self,
        plugin: ChannelPlugin,
        agent_invoker: "AgentInvoker",
        session_resolver: "SessionResolver",
        tenant_resolver: "TenantResolver",
        channel_manager: ChannelManager,
    ):
        self._plugin = plugin
        self._agent_invoker = agent_invoker
        self._session_resolver = session_resolver
        self._tenant_resolver = tenant_resolver
        self._channel_manager = channel_manager

    async def process_inbound(
        self,
        raw_message: Any,
        account_config: ChannelAccountConfig,
    ) -> AsyncIterator[tuple[ReplyPayload, EventKind]]:
        """Process an inbound message through the composed adapter pipeline.

        Yields (payload, kind) tuples. The caller decides transport encoding.
        For sync channels (web): iterate and stream.
        For async channels (telegram): collect in background task, deliver via outbound adapter.
        """

        # ── Stage 1: Normalize (plugin-specific) ───────────────
        # This is the ONLY stage that's channel-specific and required.
        # The normalizer produces a fully-populated NormalizedMessage.
        message = await self._normalize(raw_message, account_config)

        # ── Stage 2: Security check (optional adapter) ─────────
        if self._plugin.security:
            dm_policy = self._plugin.security.resolve_dm_policy(
                account=self._plugin.config.resolve_account(account_config),
                account_config=account_config,
            )
            if dm_policy and not self._is_allowed(message, dm_policy):
                return  # Silently drop or queue for approval

        # ── Stage 3: Strip mentions (optional adapter) ──────────
        body = message.body_for_agent
        if self._plugin.mentions:
            body = self._plugin.mentions.strip_mentions(
                body, message, account_config
            )
            # Rebuild message with stripped body (immutable, so replace)
            message = dataclasses.replace(message, body_for_agent=body)

        # ── Stage 4: Command handling (optional adapter) ────────
        if self._plugin.commands:
            command = self._plugin.commands.recognize_command(message)
            if command:
                reply = await self._plugin.commands.handle_command(
                    command, message, account_config=account_config
                )
                if reply:
                    yield (reply, EventKind.FINAL)
                    return  # Command handled, don't invoke agent

        # ── Stage 5: Group policy check (optional adapter) ──────
        if self._plugin.groups and message.chat_type in (ChatType.GROUP, ChatType.CHANNEL):
            require_mention = self._plugin.groups.resolve_require_mention(
                group_id=message.sender.platform_chat_id,
                account_config=account_config,
            )
            if require_mention and not message.was_mentioned:
                return  # Not mentioned in group, ignore

        # ── Stage 6: Invoke agent ───────────────────────────────
        agent_config = self._build_agent_config(message)

        async for event in self._agent_invoker.invoke(
            session_id=message.session_id,
            config=agent_config,
            message=message,
        ):
            kind = self._classify_event(event)
            payload = self._event_to_payload(event)
            if payload:
                yield (payload, kind)

    async def _normalize(
        self, raw_message: Any, account_config: ChannelAccountConfig
    ) -> NormalizedMessage:
        """Channel-specific normalization. Fully populates the message.

        Unlike the original plan's progressive mutation, the normalizer
        resolves tenant and session upfront so the message is immutable after creation.
        """
        # Plugin provides a raw normalizer
        partial = await self._plugin.config.normalize_raw(raw_message, account_config)

        # Resolve tenant and session (common infrastructure)
        org_id = await self._tenant_resolver.resolve(partial.sender)
        session_id = await self._session_resolver.resolve(partial.sender, org_id)

        # Return fully populated, immutable message
        return dataclasses.replace(partial, org_id=org_id, session_id=session_id)

    def _is_allowed(self, message: NormalizedMessage, policy: DmPolicy) -> bool:
        if policy.policy == "everyone":
            return True
        if policy.allow_from is None:
            return False
        user_id = message.sender.platform_user_id
        if policy.normalize_entry:
            user_id = policy.normalize_entry(user_id)
        return user_id in [
            policy.normalize_entry(e) if policy.normalize_entry else e
            for e in policy.allow_from
        ]

    def _build_agent_config(self, message: NormalizedMessage) -> dict[str, Any]:
        config = {"mode": message.metadata.get("mode", "execution")}
        if message.metadata.get("mode_config"):
            config["mode_config"] = message.metadata["mode_config"]
        # Include channel capabilities so agent can adapt
        config["channel_capabilities"] = dataclasses.asdict(
            self._plugin.descriptor.capabilities
        )
        # Include channel-specific agent hints
        if self._plugin.agent_prompt:
            config["channel_hints"] = self._plugin.agent_prompt.message_tool_hints(
                account_config=ChannelAccountConfig(...)  # resolved from context
            )
        return config

    def _classify_event(self, event: Any) -> EventKind:
        """Classify an agent event into tool/block/final."""
        # Implementation depends on your event types
        event_type = getattr(event, "type", "")
        if event_type in ("tool_call_start", "tool_call_end", "tool_result"):
            return EventKind.TOOL
        if event_type == "text_message_content" and not getattr(event, "is_final", False):
            return EventKind.BLOCK
        return EventKind.FINAL

    def _event_to_payload(self, event: Any) -> ReplyPayload | None:
        text = getattr(event, "text", None) or getattr(event, "content", None)
        if text:
            return ReplyPayload(text=str(text))
        return None
```

---

## Async vs Sync Channel Handling

Channels have fundamentally different transport requirements. This is made explicit.

```python
class ChannelTransportMode(str, Enum):
    """How this channel receives and delivers messages.

    SYNC: Caller consumes the pipeline output as a stream (web SSE/WS).
    ASYNC: Webhook returns immediately, pipeline runs in background, delivers via outbound adapter.
    """
    SYNC = "sync"       # Web — caller iterates pipeline output
    ASYNC = "async"     # Telegram, Discord, Slack — background processing


# In ChannelDescriptor:
@dataclass(frozen=True)
class ChannelDescriptor:
    # ... other fields ...
    transport_mode: ChannelTransportMode = ChannelTransportMode.ASYNC
```

### Sync transport (Web)

```python
# SSE endpoint — iterates pipeline output directly
async def web_sse_endpoint(request: Request, plugin: WebChannelPlugin, pipeline: MessagePipeline):
    async def stream():
        async for payload, kind in pipeline.process_inbound(raw_input, account_config):
            # Web gets ALL events — encode as SSE
            yield encode_sse(payload, kind)

    return StreamingResponse(stream(), media_type="text/event-stream")
```

### Async transport (Telegram, Discord, etc.)

```python
# Webhook — returns 200 immediately, processes in background
async def telegram_webhook(request: Request, plugin: TelegramPlugin, pipeline: MessagePipeline):
    raw_message = await request.json()

    async def _process():
        dispatcher = ReplyDispatcher(
            deliver=lambda payload, kind: _deliver_telegram(plugin, payload, kind, account_config),
            human_delay_min_ms=800,
            human_delay_max_ms=2500,
        )
        async for payload, kind in pipeline.process_inbound(raw_message, account_config):
            if kind == EventKind.TOOL:
                dispatcher.send_tool_result(payload)
            elif kind == EventKind.BLOCK:
                dispatcher.send_block_reply(payload)
            else:
                dispatcher.send_final_reply(payload)
        await dispatcher.wait_for_idle()
        await dispatcher.close()

    asyncio.create_task(_process())
    return {"ok": True}


async def _deliver_telegram(plugin, payload, kind, account_config):
    """Telegram only delivers FINAL replies."""
    if kind != EventKind.FINAL:
        return  # Tool and block events silently dropped for Telegram
    if payload.text:
        await plugin.outbound.send_text(
            to=..., text=payload.text, account_id=account_config.id
        )
```

---

## Protocols (Gateway ↔ coworker_api Boundary)

```python
class AgentInvoker(Protocol):
    """Abstracts agent invocation. Gateway defines, coworker_api implements.

    Implementation wraps OrchestratorManager + AGUIStreamProcessor.
    Yields raw BaseEvent objects (no SSE encoding — that's transport's job)."""

    async def invoke(
        self,
        session_id: str,
        config: dict[str, Any],
        message: NormalizedMessage,
    ) -> AsyncIterator[Any]:
        """Invoke the agent and stream events."""
        ...


class SessionResolver(Protocol):
    """Maps ChannelIdentity -> internal session_id. Creates if needed."""

    async def resolve(self, identity: ChannelIdentity, org_id: str) -> str: ...


class TenantResolver(Protocol):
    """Resolves org_id from ChannelIdentity.
    Web: from auth headers. Webhooks: from channel account config."""

    async def resolve(self, identity: ChannelIdentity) -> str: ...


class ChannelAccountRepo(Protocol):
    """CRUD for channel account configurations. DB-backed."""

    async def list_active(self) -> list[ChannelAccountConfig]: ...
    async def get_by_id(self, account_id: str) -> ChannelAccountConfig | None: ...
    async def get_by_type_and_org(self, channel_type: str, org_id: str) -> list[ChannelAccountConfig]: ...
    async def create(self, config: ChannelAccountConfig) -> ChannelAccountConfig: ...
    async def update(self, config: ChannelAccountConfig) -> ChannelAccountConfig: ...
    async def delete(self, account_id: str) -> None: ...
```

---

## Channel Registry

```python
class ChannelRegistry:
    """Registry pattern with two-tier access.

    OpenClaw equivalent: PluginRegistry.channels (src/plugins/registry.ts:124-138)
    + ChannelDock lookup (src/channels/dock.ts:421-453).

    Light access (descriptors): always available, no heavy imports.
    Heavy access (plugins): loaded on demand when starting accounts."""

    def __init__(self):
        self._descriptors: dict[str, ChannelDescriptor] = {}
        self._plugins: dict[str, ChannelPlugin] = {}
        self._aliases: dict[str, str] = {}

    def register(self, plugin: ChannelPlugin) -> None:
        channel_id = plugin.descriptor.id
        self._descriptors[channel_id] = plugin.descriptor
        self._plugins[channel_id] = plugin
        for alias in plugin.descriptor.aliases:
            self._aliases[alias] = channel_id

    def get_descriptor(self, channel_type: str) -> ChannelDescriptor | None:
        """Light access — no heavy imports."""
        resolved = self._aliases.get(channel_type, channel_type)
        return self._descriptors.get(resolved)

    def get(self, channel_type: str) -> ChannelPlugin | None:
        """Heavy access — full plugin with platform SDK."""
        resolved = self._aliases.get(channel_type, channel_type)
        return self._plugins.get(resolved)

    def list_descriptors(self) -> list[ChannelDescriptor]:
        return sorted(self._descriptors.values(), key=lambda d: d.order)

    def all_plugins(self) -> list[ChannelPlugin]:
        return list(self._plugins.values())

    def normalize_channel_id(self, raw: str) -> str | None:
        key = raw.strip().lower()
        if key in self._descriptors:
            return key
        return self._aliases.get(key)
```

---

## Gateway — Central Hub

```python
class Gateway:
    """Central gateway hub. Manages lifecycle, provides lookups, routes completions.

    OpenClaw equivalent: startGatewayServer (src/gateway/server.impl.ts:147+)
    + ChannelManager + AgentEventHandler."""

    def __init__(
        self,
        registry: ChannelRegistry,
        channel_manager: ChannelManager,
        account_repo: ChannelAccountRepo,
    ):
        self._registry = registry
        self._channel_manager = channel_manager
        self._account_repo = account_repo

    async def start(self) -> None:
        """Boot all active channel accounts."""
        await self._channel_manager.start_all()

    async def stop(self) -> None:
        """Graceful shutdown."""
        await self._channel_manager.stop_all()

    def get_channel(self, channel_type: str) -> ChannelPlugin | None:
        return self._registry.get(channel_type)

    def get_descriptor(self, channel_type: str) -> ChannelDescriptor | None:
        return self._registry.get_descriptor(channel_type)

    def get_runtime_snapshot(self) -> dict[str, dict[str, ChannelAccountSnapshot]]:
        return self._channel_manager.get_runtime_snapshot()

    async def start_account(self, channel_type: str, account_id: str) -> None:
        await self._channel_manager.start_account(channel_type, account_id)

    async def stop_account(self, channel_type: str, account_id: str) -> None:
        await self._channel_manager.stop_account(channel_type, account_id)

    async def handle_completion(
        self, channel_identity: ChannelIdentity, message: str
    ) -> None:
        """Route an execution completion back to the originating channel."""
        plugin = self._registry.get(channel_identity.channel_type)
        if not plugin:
            return
        account_config = await self._account_repo.get_by_id(channel_identity.account_id)
        if not account_config:
            return
        payload = ReplyPayload(text=message)
        await plugin.outbound.send_text(
            to=channel_identity.platform_chat_id or channel_identity.platform_user_id,
            text=message,
            account_id=channel_identity.account_id,
        )
```

---

## Web Channel Implementation

```python
# ── Descriptor (lightweight) ────────────────────────────────────

WEB_DESCRIPTOR = ChannelDescriptor(
    id="web",
    label="Web",
    docs_path="/channels/web",
    blurb="Browser-based chat interface with SSE and WebSocket support.",
    capabilities=ChannelCapabilities(
        chat_types=(ChatType.DIRECT,),
        media=True,
        rich_formatting=True,
        block_streaming=True,
        max_text_length=100_000,
    ),
    transport_mode=ChannelTransportMode.SYNC,
    debounce_ms=0,
)


# ── Plugin ──────────────────────────────────────────────────────

class WebChannelPlugin:
    descriptor = WEB_DESCRIPTOR

    config = WebConfigAdapter()
    outbound = WebOutboundAdapter()

    # Optional adapters — web doesn't need most of them
    gateway = None          # No persistent connections to manage
    security = None         # Auth handled at HTTP middleware layer
    pairing = None
    status = None
    mentions = None         # No @mentions in web chat
    groups = None           # No group chat in web
    threading = None
    messaging = None
    directory = None
    commands = None         # No /commands in web
    actions = None
    auth = None             # Web auth is JWT, handled by FastAPI deps
    onboarding = None
    agent_prompt = None


class WebConfigAdapter:
    def list_account_ids(self, org_id: str) -> list[str]:
        return ["default"]

    def resolve_account(self, account_config: ChannelAccountConfig) -> dict:
        return account_config.config

    def is_enabled(self, account: Any) -> bool:
        return True

    def is_configured(self, account: Any) -> bool:
        return True

    def describe_account(self, account: Any) -> ChannelAccountSnapshot:
        return ChannelAccountSnapshot(account_id="default", enabled=True, configured=True)

    async def normalize_raw(
        self, raw: "RunAgentInput", account_config: ChannelAccountConfig
    ) -> NormalizedMessage:
        """Web normalizes from AG-UI RunAgentInput."""
        return NormalizedMessage(
            id=raw.run_id or str(uuid.uuid4()),
            sender=ChannelIdentity(
                channel_type="web",
                account_id=account_config.id,
                platform_user_id=raw.metadata.get("user_id", "anonymous"),
                platform_chat_id=None,
                display_name=None,
                metadata=raw.metadata or {},
            ),
            chat_type=ChatType.DIRECT,
            direction="inbound",
            body=raw.messages[-1].content if raw.messages else "",
            body_for_agent=raw.messages[-1].content if raw.messages else "",
            body_for_commands=raw.messages[-1].content if raw.messages else "",
            content=[ContentItem(type=ContentType.TEXT, text=raw.messages[-1].content)]
                if raw.messages else [],
            org_id="",          # Filled by pipeline
            session_id="",      # Filled by pipeline
            thread_id=raw.thread_id,
            reply_to_id=None,
            thread_label=None,
            message_thread_id=None,
            sender_name=None,
            sender_id=raw.metadata.get("user_id"),
            sender_username=None,
            timestamp=datetime.utcnow(),
            was_mentioned=False,
            originating_channel="web",
            originating_to=None,
            metadata={
                "mode": raw.metadata.get("mode"),
                "mode_config": raw.metadata.get("mode_config"),
                "forwarded_props": raw.forwarded_props,
            },
        )


class WebOutboundAdapter:
    """Web outbound is a no-op — SSE/WS transport handles delivery."""
    delivery_mode = DeliveryMode.DIRECT
    text_chunk_limit = 100_000
    chunker_mode = "text"

    async def send_text(self, to, text, **kwargs):
        # For async completions, this could push via WebSocket
        return OutboundDeliveryResult(channel="web", message_id=str(uuid.uuid4()))

    async def send_media(self, to, text, media_url, **kwargs):
        return OutboundDeliveryResult(channel="web", message_id=str(uuid.uuid4()))

    async def send_payload(self, to, payload, **kwargs):
        return OutboundDeliveryResult(channel="web", message_id=str(uuid.uuid4()))
```

---

## Database Models

```python
# coworker_api/gateway/models.py

class ChannelAccount(Base):
    """Stores channel account configurations per tenant."""
    __tablename__ = "channel_accounts"

    id: Mapped[str] = mapped_column(primary_key=True, default=lambda: str(uuid.uuid4()))
    org_id: Mapped[str] = mapped_column(ForeignKey("organizations.id"), index=True)
    channel_type: Mapped[str] = mapped_column(String(50))
    name: Mapped[str] = mapped_column(String(255))
    enabled: Mapped[bool] = mapped_column(default=True)
    status: Mapped[str] = mapped_column(String(50), default="active")
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    credentials: Mapped[dict] = mapped_column(JSONB, default=dict)  # Encrypted at rest
    created_at: Mapped[datetime] = mapped_column(default=func.now())
    updated_at: Mapped[datetime] = mapped_column(default=func.now(), onupdate=func.now())

    __table_args__ = (
        Index("ix_channel_accounts_org_type", "org_id", "channel_type"),
    )


class ChannelSession(Base):
    """Maps platform identities to internal sessions. Enables session continuity
    when a user messages from the same platform identity."""
    __tablename__ = "channel_sessions"

    id: Mapped[str] = mapped_column(primary_key=True, default=lambda: str(uuid.uuid4()))
    channel_type: Mapped[str] = mapped_column(String(50))
    account_id: Mapped[str] = mapped_column(ForeignKey("channel_accounts.id"), index=True)
    platform_user_id: Mapped[str] = mapped_column(String(255))
    platform_chat_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    session_id: Mapped[str] = mapped_column(ForeignKey("sessions.id"), index=True)
    org_id: Mapped[str] = mapped_column(ForeignKey("organizations.id"), index=True)
    is_active: Mapped[bool] = mapped_column(default=True)
    last_message_at: Mapped[datetime | None] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(default=func.now())

    __table_args__ = (
        UniqueConstraint("channel_type", "account_id", "platform_user_id", "platform_chat_id",
                         name="uq_channel_session_identity"),
    )


# Modified: Execution model
# Add to coworker_api/executions/models.py:
# source_channel: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
# Stores serialized ChannelIdentity for completion routing
```

---

## Completion Handler (Redis Streams, Not Pub/Sub)

```python
class ExecutionCompletionHandler:
    """Routes execution completions back to originating channels.

    Uses Redis Streams (not pub/sub) for at-least-once delivery with consumer groups.
    Multiple gateway instances can consume without duplicates."""

    STREAM = "gateway:execution_completed"
    GROUP = "gateway-consumers"

    def __init__(self, redis: Redis, gateway: Gateway):
        self._redis = redis
        self._gateway = gateway

    async def start(self) -> None:
        # Create consumer group (idempotent)
        try:
            await self._redis.xgroup_create(self.STREAM, self.GROUP, id="0", mkstream=True)
        except Exception:
            pass  # Group already exists

        self._task = asyncio.create_task(self._listen())

    async def _listen(self) -> None:
        consumer_id = f"gateway-{uuid.uuid4().hex[:8]}"
        while True:
            try:
                entries = await self._redis.xreadgroup(
                    self.GROUP, consumer_id, {self.STREAM: ">"}, count=10, block=5000
                )
                for stream, messages in entries:
                    for msg_id, data in messages:
                        try:
                            await self._handle(data)
                            await self._redis.xack(self.STREAM, self.GROUP, msg_id)
                        except Exception:
                            pass  # Will be retried (pending entry list)
            except Exception:
                await asyncio.sleep(1)

    async def _handle(self, data: dict) -> None:
        identity = ChannelIdentity(**json.loads(data[b"source_channel"]))
        text = data[b"message"].decode()
        await self._gateway.handle_completion(identity, text)
```

---

## Package Structure

```
packages/gateway/
  pyproject.toml
  src/gateway/
    __init__.py
    core/
      __init__.py
      types.py              # All data types (NormalizedMessage, ChannelIdentity, etc.)
      gateway.py            # Gateway class
      registry.py           # ChannelRegistry
      manager.py            # ChannelManager (lifecycle, state)
      dispatcher.py         # ReplyDispatcher
      pipeline.py           # MessagePipeline (composable adapter calls)
    protocols.py            # AgentInvoker, SessionResolver, TenantResolver, ChannelAccountRepo
    exceptions.py           # ChannelNotFoundError, SecurityDeniedError, etc.
    channels/
      __init__.py
      descriptors.py        # All ChannelDescriptor definitions (lightweight)
    web/
      __init__.py
      plugin.py             # WebChannelPlugin + adapters
      transports.py         # SSETransport, WebSocketTransport

packages/coworker_api/
  src/coworker_api/
    gateway/
      __init__.py
      models.py             # ChannelAccount, ChannelSession DB models
      repository.py         # ChannelAccountRepository, ChannelSessionRepository
      agent_invoker.py      # AgentInvoker impl
      session_resolver.py   # SessionResolver impl
      tenant_resolver.py    # TenantResolver impl
      setup.py              # Gateway initialization + wiring
      completion_handler.py # Redis Streams completion listener
      router.py             # FastAPI routes
```

---

## Key Differences from Original Plan

| Original Plan | This Design | Why |
|---|---|---|
| Template Method (fixed pipeline) | Composable adapters + MessagePipeline | Channels that don't need security/mentions/commands skip them entirely |
| `ChannelPlugin` as ABC with abstract methods | `ChannelPlugin` as Protocol with optional adapter slots | Channels implement only what they need |
| One plugin instance per channel type | One plugin definition, many accounts via ChannelManager | Multi-tenant: 50 orgs each with their own Telegram bot |
| `NormalizedMessage` mutated during pipeline | Immutable after creation (frozen dataclass) | No half-populated messages; type system tells the truth |
| `ResponseFilter` (filtering + formatting) | `ReplyDispatcher` with `EventKind` discrimination | Separates filtering (by kind) from formatting (by outbound adapter) |
| `start(config)` / `stop()` on plugin instance | `start_account(ctx)` / `stop_account(ctx)` with external state | Per-account lifecycle with cancel signals |
| No capabilities | `ChannelCapabilities` on descriptor | Agent knows what the channel supports |
| No debouncing | `debounce_ms` on descriptor | Prevent rapid-fire message spam |
| No mention stripping | `MentionAdapter` | Clean message before agent sees it |
| No command handling | `CommandAdapter` | /start, /help short-circuit before agent |
| No health/status model | `StatusAdapter` + `ChannelAccountSnapshot` | Production monitoring |
| Redis pub/sub for completions | Redis Streams with consumer groups | At-least-once delivery, multi-instance safe |
| `handle_message() -> AsyncIterator` always | Explicit `ChannelTransportMode.SYNC` vs `ASYNC` | Web streams inline; webhooks return 200 and process in background |
| `to_outbound` wraps event in metadata | `OutboundAdapter` with `send_text`/`send_media` | Clean separation of serialization and delivery |
| `build_agent_config` returns untyped dict | Agent config includes capabilities + channel hints | Agent adapts to channel constraints |
| No threading model | `ThreadingAdapter` per channel | Discord threads, Slack threads, Telegram reply chains |
| No group policy | `GroupAdapter` | Require @mention in groups, restrict tools |
| No directory | `DirectoryAdapter` | List contacts/groups for outbound targeting |

---

## Walkthrough: How It All Fits Together

### The Big Picture

The gateway is the **bridge between messaging channels** (web chat, Telegram, Discord, email, etc.) and **the AI agent**. In a multi-tenant SaaS context, it handles the fact that 50 different organizations might each have their own Telegram bot, their own Discord server, their own web widget — all running through the same gateway process.

### Why Each Principle Matters

**Stateless Plugins, External State** — A plugin (e.g., the Telegram plugin) has zero instance variables. It's just a bag of functions. All mutable state (is this account running? when did it last connect? what's the error?) lives in the `ChannelManager`. The plugin gets `get_status()` and `set_status()` callbacks passed to it. If the plugin crashes, the state survives. You can restart an account without losing bookkeeping.

**Composable Optional Adapters** — Instead of a base class with 20 abstract methods you must override, the plugin is a struct with ~20 optional slots. The Telegram plugin implements `gateway`, `outbound`, `security`, `mentions`, `commands`. The web plugin implements just `config` and `outbound`. The infrastructure checks `if plugin.security:` before calling it. No dead code — a channel that doesn't have groups doesn't need to implement `resolve_require_mention()` returning `None`.

**Two-Tier Weight (Descriptor vs Plugin)** — `ChannelDescriptor` is lightweight metadata (name, capabilities, chunk limits) that's always loaded. `ChannelPlugin` is the heavy implementation with platform SDK imports. Routing logic, allowlist checks, and UI rendering only need descriptors. You don't want to import the Telegram SDK just to check if a channel supports threads.

**Per-Account Lifecycle** — One plugin definition, many accounts. If 3 tenants each have a Telegram bot, the `ChannelManager` runs `start_account()` three times on the same plugin, each with its own cancel signal, task, and snapshot.

**Reply Dispatcher with Kind Discrimination** — Every outbound message is classified as `tool` (internal debugging), `block` (streaming text mid-generation), or `final` (complete reply). External channels like Telegram only receive finals. WebSocket clients get everything. Human-like delays (800-2500ms) are inserted between blocks.

**Config-Driven Multi-Tenancy** — Plugins receive a `TenantConfig` scoped to an org. The plugin code never changes between tenants — isolation happens at the config boundary.

**Capabilities Declaration** — Each channel declares what it supports (threads? reactions? media? inline streaming?). The agent receives this so it doesn't try to send polls on a channel that can't render them.

### Core Data Flow

Here's how a message flows through the system:

```
User sends message on Telegram
        |
        v
   Webhook hits FastAPI endpoint
        |
        v
   Returns 200 immediately (async transport)
        |
        v
   MessagePipeline.process_inbound() runs in background task:
        |
        +-- Stage 1: normalize_raw()
        |   Telegram Update -> NormalizedMessage (frozen/immutable)
        |
        +-- Stage 2: security check
        |   Is this user allowed? (if plugin.security exists)
        |
        +-- Stage 3: strip mentions
        |   Remove "@botname" from text (if plugin.mentions exists)
        |
        +-- Stage 4: command handling
        |   "/start" short-circuits before agent (if plugin.commands exists)
        |
        +-- Stage 5: group policy
        |   Require @mention in groups? (if plugin.groups exists)
        |
        +-- Stage 6: invoke agent
            Yields tool/block/final events
                |
                v
         ReplyDispatcher (serialized queue)
                |
                +-- normalize: strip silent tokens, heartbeat tokens, sanitize
                +-- classify: tool -> drop, block -> delay, final -> deliver
                +-- deliver via OutboundAdapter.send_text()
                        |
                        v
                   BlockStreamingCoalescer
                   (buffer 800-1200 chars, flush on paragraph breaks)
                        |
                        v
                   TextChunker
                   (split long messages for platform limits)
                        |
                        v
                   Telegram Bot API sendMessage()
```

For **web chat**, the flow is different — it's **sync transport**:

```
User sends message via SSE/WebSocket
        |
        v
   pipeline.process_inbound() returns AsyncIterator
        |
        v
   FastAPI streams each (payload, kind) as SSE events directly
   (no ReplyDispatcher needed — the client handles all event kinds)
```

### ChannelManager State Model

Owns all runtime state. Three maps per channel type:

| Map | Key | Value | Purpose |
|---|---|---|---|
| `cancel_events` | `(channel_type, account_id)` | `asyncio.Event` | Signal account to stop |
| `tasks` | `(channel_type, account_id)` | `asyncio.Task` | Running coroutine |
| `snapshots` | `(channel_type, account_id)` | `ChannelAccountSnapshot` | Runtime state |

The lifecycle: `start_account()` checks enabled -> checks configured -> creates cancel event -> creates context with `get_status`/`set_status` callbacks -> calls `plugin.gateway.start_account(ctx)` in a task -> on error/cancel, updates snapshot.

### Multi-Tenancy Isolation

Plugins don't know about tenants. Tenant isolation happens at two boundaries:

1. **Config boundary** — `ChannelAccountConfig` has an `org_id`. Each tenant's Telegram bot is a separate account record in the DB.
2. **Session boundary** — `ChannelSession` maps `(channel_type, account_id, platform_user_id)` -> `session_id`, scoped to `org_id`.

When Org A's Telegram bot receives a message, it resolves to Org A's config and Org A's session — the plugin code is identical.

### Plugin Example: Minimal vs Full

A full plugin (Telegram):

```python
class TelegramPlugin:
    descriptor = TELEGRAM_DESCRIPTOR       # lightweight metadata
    config = TelegramConfigAdapter()       # resolve accounts from DB config
    outbound = TelegramOutboundAdapter()   # send_text, send_media via Bot API
    gateway = TelegramGatewayAdapter()     # start_account (webhook setup), stop_account
    security = TelegramSecurityAdapter()   # DM policy, allowlist
    mentions = TelegramMentionAdapter()    # strip @botname
    commands = TelegramCommandAdapter()    # /start, /help
    groups = TelegramGroupAdapter()        # require @mention in groups
    status = TelegramStatusAdapter()       # health probe via getWebhookInfo

    # Not needed for Telegram:
    threading = None
    directory = None
    actions = None
    auth = None
    # ... etc
```

A minimal plugin (web):

```python
class WebPlugin:
    descriptor = WEB_DESCRIPTOR
    config = WebConfigAdapter()
    outbound = WebOutboundAdapter()        # no-op — SSE handles delivery
    # everything else = None
```

### Completion Routing (Async Agents)

When an agent finishes a long-running task (minutes later), the result needs to get back to the right channel. This uses **Redis Streams** (not pub/sub) with consumer groups:

1. Agent completes -> writes to `gateway:execution_completed` stream with serialized `ChannelIdentity`
2. Gateway consumer group reads it -> looks up the plugin -> calls `outbound.send_text()`
3. Consumer ACKs the message -> at-least-once delivery even across gateway restarts

### Full Outbound Pipeline (Post-Agent)

```
Agent event -> ReplyDispatcher.enqueue()
                    |
                    +-- normalize: strip [silent], [heartbeat] tokens, sanitize text
                    +-- kind check: tool events -> dropped for external channels
                    +-- human delay: 800-2500ms between block replies
                    |
                    v
              BlockStreamingCoalescer
                    |
                    +-- buffer text until min_chars (800) reached
                    +-- flush at paragraph break if available
                    +-- hard flush at max_chars (1200)
                    +-- idle flush after 1000ms of no new text
                    |
                    v
              TextChunker
                    |
                    +-- split by platform limit (4096 for Telegram, 2000 for Discord)
                    +-- prefer paragraph -> line -> word -> hard split
                    |
                    v
              OutboundAdapter.send_text() -> Platform API
```

---

## Multi-Tenant Channel App Models

### The Two Models

When building a multi-tenant SaaS with channel integrations, there are two fundamentally different approaches for how channel apps (Slack, Discord, Teams, etc.) relate to tenants.

### Model 1: Each Tenant Creates Their Own App

Each tenant:
1. Goes to the platform developer portal (api.slack.com, Discord Developer Portal, etc.)
2. Creates their own app with their own name, avatar, permissions
3. Installs it in their workspace/server
4. Copies the bot token into your platform's settings UI
5. The gateway connects using those tokens

```
Org A -> created SlackApp-A -> installed in Workspace-A -> bot token -> gateway account "org-a"
Org B -> created SlackApp-B -> installed in Workspace-B -> bot token -> gateway account "org-b"
Org C -> created SlackApp-C -> installed in Workspace-C -> bot token -> gateway account "org-c"
```

Each "account" in the gateway is a **separate app** with its own tokens. If 3 orgs use Slack, that's 3 different Slack apps, 3 sets of tokens, 3 independent connections.

**Pros:**
- Simple implementation (no OAuth)
- Each tenant fully controls their app (name, avatar, permissions)
- No rate limit sharing between tenants

**Cons:**
- Onboarding friction — every tenant must create an app manually
- No centralized control over app permissions or updates

**This is what OpenClaw uses.** Each user creates their own Slack app and pastes tokens into config.

### Model 2: One App, Many Workspace Installs (SaaS Model)

This is how Claude Code's Slack app, Linear, Notion, and most SaaS products work:

1. **You** create ONE app (e.g., "Digital Coworker") on the platform
2. You set up OAuth with redirect URLs
3. Each tenant clicks "Add to Slack" -> OAuth consent -> platform gives you a **per-workspace bot token**
4. You store that token in your DB, keyed to the tenant's `org_id`

```
Your single Slack App "Digital Coworker"
    |
    +-- Org A clicks "Add to Slack" -> OAuth -> bot token for Workspace-A -> stored as account for org-a
    +-- Org B clicks "Add to Slack" -> OAuth -> bot token for Workspace-B -> stored as account for org-b
    +-- Org C clicks "Add to Slack" -> OAuth -> bot token for Workspace-C -> stored as account for org-c
```

**Same app identity, different bot tokens per workspace.** The app name, avatar, and permissions are identical across all installs.

**Pros:**
- Frictionless onboarding (one-click "Add to Slack")
- Centralized permission management
- Professional appearance (consistent branding)

**Cons:**
- Need OAuth implementation and token refresh logic
- Shared rate limits (Slack rate limits are per-app, not per-workspace)
- Platform review process for distribution

### Why the Gateway Design Handles Both

The gateway doesn't care which model you use. In both cases, the `ChannelAccountConfig` looks the same:

```python
# Model 1: tenant created their own app
ChannelAccountConfig(
    id="acc_123",
    org_id="org-a",
    channel_type="slack",
    config={"mode": "socket"},
    credentials={"bot_token": "xoxb-THEIR-TOKEN", "app_token": "xapp-THEIR-TOKEN"},
)

# Model 2: your app installed via OAuth
ChannelAccountConfig(
    id="acc_456",
    org_id="org-b",
    channel_type="slack",
    config={"mode": "http"},  # webhooks, not socket mode
    credentials={"bot_token": "xoxb-OAUTH-ISSUED-TOKEN"},
)
```

The plugin's `start_account()` receives the token from the config and connects. It doesn't know or care how the token was obtained.

### What Changes Between the Two Models

| Concern | Model 1 (Self-hosted apps) | Model 2 (Shared app + OAuth) |
|---|---|---|
| How tokens arrive | User pastes in settings UI | OAuth callback stores in DB |
| Onboarding flow | "Create app, copy tokens" | "Click Add to Slack" button |
| Socket Mode vs HTTP | Socket Mode (needs app token) | HTTP webhooks (one endpoint for all) |
| Webhook routing | N/A (socket) | Endpoint receives events for ALL workspaces; look up `team_id` -> `org_id` -> `account_config` |
| App identity | Different name/avatar per tenant | Same name/avatar everywhere |
| Token refresh | Manual (tokens don't expire) | Need OAuth token rotation |

### Recommended Approach for Multi-Tenant SaaS

Use **Model 2** (one app, OAuth installs). Add the following to the gateway:

**1. AuthAdapter for OAuth flow:**

```python
class SlackAuthAdapter:
    async def get_install_url(self, org_id: str) -> str:
        """Generate Slack OAuth URL with state=org_id."""
        ...

    async def handle_oauth_callback(self, code: str, state: str) -> ChannelAccountConfig:
        """Exchange code for bot token, store as new channel account."""
        ...
```

**2. Shared webhook endpoint with tenant routing:**

```python
@router.post("/hooks/slack/events")
async def slack_events(request: Request):
    payload = await request.json()
    team_id = payload.get("team_id")
    # Look up which org owns this workspace
    account = await account_repo.get_by_platform_id("slack", team_id)
    # Route to pipeline with that account's config
    await pipeline.process_inbound(payload, account)
```

**3. Everything else stays identical.** The `ChannelManager` starts one account per workspace install, each with its own token. The plugin, dispatcher, coalescer, chunker — all unchanged.

### Per-Channel App Model Summary

| Channel | Typical Model | Notes |
|---|---|---|
| Slack | Model 2 (OAuth) | One app, "Add to Slack" button, per-workspace tokens |
| Discord | Model 2 (OAuth) | One bot application, OAuth to add to servers |
| Microsoft Teams | Model 2 (OAuth) | One Azure AD app, installed per tenant |
| Telegram | Model 1 (per-tenant) | Each org creates their own bot via BotFather |
| WhatsApp Business | Model 2 (API) | One Business Solution Provider, per-org phone numbers |
| Email | Neither | SMTP/IMAP credentials per org, no app model |
| Web | Neither | Built-in, auth at HTTP layer per org |

---

## Session Keys and Routing

### What a Session Key Is

A session key is the string that determines **which conversation context** (memory, thread history, agent state) a message maps to. Two messages with the same session key share the same conversation. Two messages with different session keys are isolated conversations.

In OpenClaw, session keys follow this format:

```
agent:<agentId>:<channel>:<peerKind>:<peerId>
```

For example:
- `agent:main:telegram:dm:123456` — DM with Telegram user 123456, routed to agent "main"
- `agent:main:slack:group:C0123ABCD` — Slack channel C0123ABCD, routed to agent "main"
- `agent:support:discord:dm:987654321` — DM with Discord user, routed to agent "support"
- `agent:main:main` — The default "main" session (web chat, no peer context)

### DM Scope Modes

The `dmScope` setting controls how DM conversations are isolated. This is critical for multi-tenant because different tenants may want different behavior:

| Mode | Session Key | Behavior |
|---|---|---|
| `main` (default) | `agent:<agentId>:main` | All DMs share ONE conversation. User on Telegram and same user on Slack see the same history. |
| `per-peer` | `agent:<agentId>:dm:<peerId>` | Each unique user gets their own conversation, but cross-channel messages from the same person merge. |
| `per-channel-peer` | `agent:<agentId>:<channel>:dm:<peerId>` | Each user on each channel is isolated. Telegram user 123 and Slack user 123 are separate conversations. |
| `per-account-channel-peer` | `agent:<agentId>:<channel>:<accountId>:dm:<peerId>` | Fully isolated. Same user talking to two different Telegram bots (two accounts) gets separate conversations. |

For a multi-tenant SaaS, the recommended default is **`per-account-channel-peer`** — full isolation prevents any data leakage between tenants, accounts, or channels.

### Identity Linking

OpenClaw supports linking identities across channels so that the same person on different platforms can share a conversation (when dmScope allows it):

```python
# Config: identity_links maps a canonical name to platform-prefixed peer IDs
identity_links = {
    "alice": ["telegram:123456", "slack:U0ABC"],
    "bob": ["discord:987654321", "whatsapp:+15551234567"],
}
```

When `dmScope` is `per-peer` and Alice messages from Telegram (peer ID `telegram:123456`), the system looks up the canonical name "alice" and uses that as the peerId in the session key. When she messages from Slack, it resolves to the same canonical name — same session key, same conversation.

For multi-tenant, identity links are stored **per org** in the DB:

```python
@dataclass
class IdentityLink:
    org_id: str
    canonical_name: str               # "alice"
    platform_ids: list[str]           # ["telegram:123456", "slack:U0ABC"]
```

### Binding Priority (Agent Routing)

When a message arrives, the system needs to decide **which agent** handles it. OpenClaw uses a binding priority chain that matches from most specific to least specific:

```
1. binding.peer          — Exact peer match (e.g., "route user X to agent support")
2. binding.peer.parent   — Thread parent match (thread inherits parent's binding)
3. binding.guild         — Discord server / Slack workspace match
4. binding.team          — Team-level match
5. binding.account       — Channel account match (e.g., "all messages to this Telegram bot go to agent sales")
6. binding.channel       — Channel type match with wildcard account (e.g., "all Telegram -> agent main")
7. default               — Falls back to the default agent
```

For multi-tenant, bindings are stored per org:

```python
@dataclass(frozen=True)
class AgentBinding:
    """Routes messages to specific agents based on match criteria.

    OpenClaw equivalent: routing bindings (src/routing/bindings.ts)."""
    org_id: str
    agent_id: str
    match: BindingMatch

@dataclass(frozen=True)
class BindingMatch:
    channel: str                      # Required: which channel
    account_id: str | None = None     # Optional: specific account (None = default, "*" = any)
    peer: RoutePeer | None = None     # Optional: specific peer
    guild_id: str | None = None       # Optional: Discord server ID
    team_id: str | None = None        # Optional: Slack workspace ID
```

### Thread Session Keys

When a channel supports threads (Slack, Discord), the session key gets a thread suffix:

```
base:    agent:main:slack:group:C0123ABCD
thread:  agent:main:slack:group:C0123ABCD:thread:1234567890.123456
```

The thread inherits the parent's agent binding but gets its own conversation history. This is handled by `resolveThreadSessionKeys()`:

```python
def resolve_thread_session_key(
    base_session_key: str,
    thread_id: str | None,
) -> tuple[str, str | None]:
    """Append thread suffix to session key if thread_id is present.

    Returns (session_key, parent_session_key).

    OpenClaw equivalent: resolveThreadSessionKeys (session-key.ts:233-249)."""
    if not thread_id:
        return (base_session_key, None)
    thread_key = f"{base_session_key}:thread:{thread_id.lower()}"
    return (thread_key, base_session_key)
```

### Group History Keys

Groups and channels get their own history key format that includes the account:

```
<channel>:<accountId>:<peerKind>:<peerId>
```

Example: `slack:default:group:C0123ABCD`

This ensures that group message history is scoped to the specific account and channel, which is important when the same group could theoretically be visible through multiple accounts.

### How Session Resolution Works in the Pipeline

The `SessionResolver` protocol bridges the gap between channel identities and internal session keys:

```python
class SessionResolver(Protocol):
    async def resolve(self, identity: ChannelIdentity, org_id: str) -> str:
        """Maps a channel identity to an internal session key.

        Steps:
        1. Load org's routing config (bindings, dmScope, identityLinks)
        2. Determine peer kind (dm/group/channel) from identity
        3. Check bindings in priority order to find the target agent
        4. Build session key using agentId + channel + accountId + peer + dmScope
        5. If thread_id present, append thread suffix
        6. Upsert ChannelSession record in DB for continuity
        7. Return the session key
        """
        ...
```

The concrete implementation in `coworker_api`:

```python
class DatabaseSessionResolver:
    def __init__(self, session_repo: ChannelSessionRepository, config_repo: OrgConfigRepository):
        self._session_repo = session_repo
        self._config_repo = config_repo

    async def resolve(self, identity: ChannelIdentity, org_id: str) -> str:
        # 1. Load org routing config
        org_config = await self._config_repo.get(org_id)
        dm_scope = org_config.session.dm_scope or "per-account-channel-peer"
        identity_links = org_config.session.identity_links or {}
        bindings = org_config.routing.bindings or []

        # 2. Determine peer
        peer = RoutePeer(
            kind="group" if identity.platform_chat_id else "dm",
            id=identity.platform_chat_id or identity.platform_user_id,
        )

        # 3. Resolve agent route (binding priority chain)
        route = resolve_agent_route(
            bindings=bindings,
            channel=identity.channel_type,
            account_id=identity.account_id,
            peer=peer,
        )

        # 4. Build session key
        session_key = build_session_key(
            agent_id=route.agent_id,
            channel=identity.channel_type,
            account_id=identity.account_id,
            peer=peer,
            dm_scope=dm_scope,
            identity_links=identity_links,
        )

        # 5. Thread suffix
        thread_id = identity.metadata.get("thread_id")
        if thread_id:
            session_key = f"{session_key}:thread:{thread_id.lower()}"

        # 6. Upsert channel session for continuity
        await self._session_repo.upsert(
            channel_type=identity.channel_type,
            account_id=identity.account_id,
            platform_user_id=identity.platform_user_id,
            platform_chat_id=identity.platform_chat_id,
            session_key=session_key,
            org_id=org_id,
        )

        return session_key
```

### Multi-Tenant Session Key Scoping

In OpenClaw (single-tenant), session keys don't include org_id because there's only one tenant. In multi-tenant, tenant isolation is enforced at two levels:

1. **Account-level isolation** — Each `ChannelAccountConfig` belongs to an `org_id`. Different orgs have different accounts, so their session keys naturally differ (the `accountId` segment is different).

2. **DB-level isolation** — The `ChannelSession` table includes `org_id`, so even if two session keys happened to collide, the DB query is always scoped: `WHERE org_id = ?`.

3. **Agent-level isolation** — Each org has its own agent definitions. The `agentId` in the session key comes from the org's routing config, not a global pool.

This means you do NOT need to prefix session keys with `org_id` — the natural account scoping already provides isolation. But as a safety net, you can use `per-account-channel-peer` as the default dmScope, which includes the accountId in the key.
