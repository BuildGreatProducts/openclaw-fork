# ClawHub - Comprehensive Build Plan

## Overview

ClawHub is a web app that Openclaw users connect to via a native Openclaw plugin, giving them full visibility into all agents running on their setup, what they're doing, AI usage costs, session history, and more.

**Stack:** Next.js + Convex + Clerk + Polar
**Connectivity:** Native Openclaw plugin pushes telemetry to Convex
**Sync model:** Periodic sync (configurable interval, default 60s) + on-demand
**Monetization:** Open source repo + hosted version with flat subscription via Polar
**Target:** Individual users first, data model designed to support teams later

---

## Architecture

```
┌─────────────────────┐        HTTPS POST         ┌──────────────────────────┐
│  User's Machine      │  ─────────────────────►  │  ClawHub (Hosted)         │
│                      │   (every 60s + events)    │                           │
│  ┌────────────────┐  │                           │  ┌─────────────────────┐  │
│  │  Openclaw       │  │                           │  │  Convex Backend      │  │
│  │  Gateway        │  │                           │  │                     │  │
│  │  ┌────────────┐│  │   Telemetry payloads      │  │  HTTP Actions       │  │
│  │  │ ClawHub    ││──│──────────────────────────►│  │  (ingest endpoint)  │  │
│  │  │ Plugin     ││  │                           │  │       │             │  │
│  │  └────────────┘│  │                           │  │       ▼             │  │
│  │                │  │                           │  │  Mutations          │  │
│  │  Hooks:        │  │                           │  │  (write to tables)  │  │
│  │  - llm_output  │  │                           │  │       │             │  │
│  │  - session_*   │  │                           │  │       ▼             │  │
│  │  - agent_end   │  │                           │  │  Queries            │  │
│  │  - tool calls  │  │                           │  │  (read for UI)      │  │
│  └────────────────┘  │                           │  └─────────────────────┘  │
│                      │                           │                           │
└─────────────────────┘                           │  ┌─────────────────────┐  │
                                                   │  │  Next.js Frontend    │  │
                                                   │  │                     │  │
                                                   │  │  Clerk (Auth)       │  │
                                                   │  │  Polar (Billing)    │  │
                                                   │  │  ClawHub UI         │  │
                                                   │  └─────────────────────┘  │
                                                   └──────────────────────────┘
```

---

## Part 1: Openclaw Plugin (`openclaw-clawhub-plugin`)

This is a native Openclaw plugin that runs inside the user's Openclaw gateway and pushes telemetry data to the hosted Convex backend.

### 1.1 Plugin Manifest

```
openclaw-clawhub-plugin/
├── openclaw.plugin.json
├── src/
│   ├── index.ts          # Plugin entry point
│   ├── collector.ts      # Event collection & batching
│   ├── pusher.ts         # HTTP push to Convex
│   ├── snapshot.ts       # Periodic full-state snapshots
│   └── types.ts          # Shared telemetry types
├── skills/
│   └── clawhub/
│       └── SKILL.md      # User-invocable skill for status/config
├── package.json
└── tsconfig.json
```

**openclaw.plugin.json:**
```json
{
  "id": "openclaw-clawhub",
  "name": "ClawHub",
  "description": "Connect your Openclaw to ClawHub for monitoring, analytics, and cost tracking",
  "version": "1.0.0",
  "kind": "telemetry",
  "skills": ["./skills/clawhub"],
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": { "type": "string", "description": "ClawHub API key (from your ClawHub settings)" },
      "endpoint": { "type": "string", "default": "https://<convex-deployment>.convex.site/ingest" },
      "syncIntervalMs": { "type": "number", "default": 60000 },
      "batchSize": { "type": "number", "default": 50 },
      "enabled": { "type": "boolean", "default": true }
    },
    "required": ["apiKey"]
  },
  "uiHints": {
    "apiKey": { "label": "ClawHub API Key", "sensitive": true },
    "endpoint": { "label": "ClawHub Endpoint URL" },
    "syncIntervalMs": { "label": "Sync Interval (ms)" }
  }
}
```

### 1.2 Plugin Event Collection

The plugin hooks into these Openclaw lifecycle events:

| Hook | Data Captured | Purpose |
|------|---------------|---------|
| `gateway_start` | port, timestamp | Instance came online |
| `gateway_stop` | reason, timestamp | Instance went offline |
| `session_start` | sessionId, agentId, resumedFrom | Track session creation |
| `session_end` | sessionId, messageCount, durationMs | Track session completion |
| `llm_output` | runId, sessionId, provider, model, usage (input/output/cache tokens) | Token usage & cost tracking |
| `agent_end` | messages, success, error, durationMs | Agent run outcomes |
| `before_tool_call` | toolName, params | Tool usage tracking |
| `after_tool_call` | toolName, result, error, durationMs | Tool performance |

### 1.3 Data Push Strategy

**Batched periodic sync (default: every 60 seconds):**
1. Events accumulate in an in-memory buffer
2. Every `syncIntervalMs`, the pusher flushes the buffer
3. Events are sent as a batch POST to the Convex HTTP action endpoint
4. On failure: retry with exponential backoff (2s, 4s, 8s), then buffer to disk
5. On gateway_stop: flush remaining events immediately

**Periodic snapshot (every 5 minutes):**
A full-state snapshot is also pushed periodically containing:
- Instance health status
- All active sessions with metadata
- Aggregate usage totals (from `sessions.usage` gateway method)
- Channel connectivity status
- Agent list and configurations
- Cron job statuses

**Payload format:**
```typescript
type TelemetryPayload = {
  instanceId: string;        // Unique per Openclaw install (derived from device identity)
  apiKey: string;            // Links to user's ClawHub account
  timestamp: number;
  kind: "events" | "snapshot";
  events?: TelemetryEvent[];
  snapshot?: InstanceSnapshot;
};

type TelemetryEvent = {
  id: string;                // UUID for deduplication
  type: "session_start" | "session_end" | "llm_output" | "agent_end" | "tool_call" | "gateway_start" | "gateway_stop";
  timestamp: number;
  agentId?: string;
  sessionId?: string;
  data: Record<string, unknown>;
};

type InstanceSnapshot = {
  health: { status: "healthy" | "degraded" | "offline"; details: Record<string, unknown> };
  agents: Array<{ id: string; name: string; isDefault: boolean }>;
  activeSessions: Array<{
    sessionId: string;
    agentId: string;
    updatedAt: number;
    inputTokens: number;
    outputTokens: number;
    totalTokens: number;
    model: string;
    provider: string;
    channel?: string;
  }>;
  usage: {
    totalTokens: number;
    totalCost: number;
    inputCost: number;
    outputCost: number;
    cacheReadCost: number;
    cacheWriteCost: number;
  };
  channels: Array<{ id: string; status: "connected" | "disconnected" | "error" }>;
  cronJobs: Array<{ id: string; name: string; enabled: boolean; lastStatus?: string; nextRunAtMs?: number }>;
};
```

### 1.4 Plugin Registration as Gateway Method

The plugin also registers a gateway method so users can check plugin status:

```typescript
api.registerGatewayMethod("clawhub.status", async (req, res, context) => {
  res(true, {
    connected: true,
    lastSyncAt: lastSyncTimestamp,
    eventsPending: eventBuffer.length,
    endpoint: config.endpoint
  });
});
```

### 1.5 User-Invocable Skill

A SKILL.md that lets users ask Openclaw about their ClawHub connection:

```markdown
---
name: clawhub
description: Check ClawHub connection status and manage settings
user-invocable: true
command-dispatch: tool
---

Check the ClawHub plugin status by calling the `clawhub.status` gateway method.
Report connection status, last sync time, and pending events.
```

---

## Part 2: Convex Backend

### 2.1 Schema Design

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  // ─── Users & Auth ───
  users: defineTable({
    clerkId: v.string(),
    email: v.string(),
    name: v.optional(v.string()),
    avatarUrl: v.optional(v.string()),
    plan: v.union(v.literal("free"), v.literal("pro")),
    polarCustomerId: v.optional(v.string()),
    polarSubscriptionId: v.optional(v.string()),
    createdAt: v.number(),
  })
    .index("by_clerk_id", ["clerkId"])
    .index("by_email", ["email"])
    .index("by_polar_customer", ["polarCustomerId"]),

  // ─── API Keys (link plugin to user account) ───
  apiKeys: defineTable({
    userId: v.id("users"),
    key: v.string(),            // hashed
    keyPrefix: v.string(),      // first 8 chars for display (e.g., "ch_ak_ab12...")
    name: v.string(),           // user-given label
    lastUsedAt: v.optional(v.number()),
    createdAt: v.number(),
    revokedAt: v.optional(v.number()),
  })
    .index("by_key", ["key"])
    .index("by_user", ["userId"]),

  // ─── Openclaw Instances ───
  instances: defineTable({
    userId: v.id("users"),
    instanceId: v.string(),     // from plugin's device identity
    name: v.optional(v.string()),
    lastSeenAt: v.number(),
    status: v.union(v.literal("online"), v.literal("offline"), v.literal("degraded")),
    healthDetails: v.optional(v.any()),
    agents: v.optional(v.array(v.object({
      id: v.string(),
      name: v.string(),
      isDefault: v.boolean(),
    }))),
    channels: v.optional(v.array(v.object({
      id: v.string(),
      status: v.string(),
    }))),
    cronJobs: v.optional(v.array(v.object({
      id: v.string(),
      name: v.string(),
      enabled: v.boolean(),
      lastStatus: v.optional(v.string()),
      nextRunAtMs: v.optional(v.number()),
    }))),
    createdAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_instance_id", ["instanceId"])
    .index("by_user_and_instance", ["userId", "instanceId"]),

  // ─── Sessions (mirrored from Openclaw) ───
  sessions: defineTable({
    instanceId: v.id("instances"),
    userId: v.id("users"),
    openclawSessionId: v.string(),
    agentId: v.optional(v.string()),
    status: v.union(v.literal("active"), v.literal("ended")),
    startedAt: v.number(),
    endedAt: v.optional(v.number()),
    durationMs: v.optional(v.number()),
    messageCount: v.optional(v.number()),
    model: v.optional(v.string()),
    provider: v.optional(v.string()),
    channel: v.optional(v.string()),
    inputTokens: v.number(),
    outputTokens: v.number(),
    totalTokens: v.number(),
    estimatedCost: v.optional(v.number()),
  })
    .index("by_instance", ["instanceId"])
    .index("by_user", ["userId"])
    .index("by_user_and_time", ["userId", "startedAt"])
    .index("by_openclaw_session", ["instanceId", "openclawSessionId"]),

  // ─── LLM Calls (individual API calls with token usage) ───
  llmCalls: defineTable({
    instanceId: v.id("instances"),
    userId: v.id("users"),
    sessionId: v.optional(v.id("sessions")),
    openclawRunId: v.string(),
    openclawSessionId: v.optional(v.string()),
    agentId: v.optional(v.string()),
    provider: v.optional(v.string()),
    model: v.optional(v.string()),
    inputTokens: v.optional(v.number()),
    outputTokens: v.optional(v.number()),
    cacheReadTokens: v.optional(v.number()),
    cacheWriteTokens: v.optional(v.number()),
    totalTokens: v.optional(v.number()),
    estimatedCost: v.optional(v.number()),
    timestamp: v.number(),
  })
    .index("by_user_and_time", ["userId", "timestamp"])
    .index("by_instance_and_time", ["instanceId", "timestamp"])
    .index("by_session", ["sessionId"]),

  // ─── Tool Calls ───
  toolCalls: defineTable({
    instanceId: v.id("instances"),
    userId: v.id("users"),
    sessionId: v.optional(v.id("sessions")),
    toolName: v.string(),
    durationMs: v.optional(v.number()),
    success: v.boolean(),
    error: v.optional(v.string()),
    timestamp: v.number(),
  })
    .index("by_user_and_time", ["userId", "timestamp"])
    .index("by_session", ["sessionId"]),

  // ─── Agent Runs ───
  agentRuns: defineTable({
    instanceId: v.id("instances"),
    userId: v.id("users"),
    sessionId: v.optional(v.id("sessions")),
    agentId: v.optional(v.string()),
    success: v.boolean(),
    error: v.optional(v.string()),
    durationMs: v.optional(v.number()),
    messageCount: v.optional(v.number()),
    timestamp: v.number(),
  })
    .index("by_user_and_time", ["userId", "timestamp"])
    .index("by_instance", ["instanceId"]),

  // ─── Daily Usage Aggregates (for fast ClawHub queries) ───
  dailyUsage: defineTable({
    userId: v.id("users"),
    instanceId: v.optional(v.id("instances")),
    date: v.string(),           // "YYYY-MM-DD"
    inputTokens: v.number(),
    outputTokens: v.number(),
    cacheReadTokens: v.number(),
    cacheWriteTokens: v.number(),
    totalTokens: v.number(),
    estimatedCost: v.number(),
    llmCallCount: v.number(),
    sessionCount: v.number(),
    agentRunCount: v.number(),
    toolCallCount: v.number(),
    errorCount: v.number(),
  })
    .index("by_user_and_date", ["userId", "date"])
    .index("by_instance_and_date", ["instanceId", "date"]),

  // ─── Processed Event IDs (deduplication) ───
  processedEvents: defineTable({
    eventId: v.string(),
    processedAt: v.number(),
  })
    .index("by_event_id", ["eventId"]),
});
```

### 2.2 Convex HTTP Actions (Ingest Endpoint)

```
convex/
├── schema.ts
├── http.ts                    # HTTP router (ingest endpoint)
├── ingest.ts                  # HTTP action: validate + fan out to mutations
├── auth.ts                    # API key validation helper
├── mutations/
│   ├── instances.ts           # Upsert instance snapshots
│   ├── sessions.ts            # Upsert sessions
│   ├── llmCalls.ts            # Insert LLM call records
│   ├── toolCalls.ts           # Insert tool call records
│   ├── agentRuns.ts           # Insert agent run records
│   ├── dailyUsage.ts          # Update daily aggregates
│   └── processedEvents.ts     # Deduplication tracking
├── queries/
│   ├── clawhub.ts             # Overview stats for ClawHub home
│   ├── instances.ts           # Instance list + detail
│   ├── sessions.ts            # Session list + detail
│   ├── usage.ts               # Usage analytics (daily, by model, etc.)
│   ├── costs.ts               # Cost breakdown queries
│   ├── agentRuns.ts           # Agent run history
│   └── toolCalls.ts           # Tool usage analytics
├── actions/
│   ├── polar.ts               # Polar webhook handler + subscription check
│   └── cleanup.ts             # Scheduled: prune old processedEvents
└── crons.ts                   # Convex cron: daily aggregate rollup, cleanup
```

**HTTP Ingest Endpoint (`convex/http.ts`):**
```typescript
import { httpRouter } from "convex/server";
import { ingestTelemetry } from "./ingest";
import { polarWebhook } from "./actions/polar";

const http = httpRouter();

http.route({
  path: "/ingest",
  method: "POST",
  handler: ingestTelemetry,
});

http.route({
  path: "/webhooks/polar",
  method: "POST",
  handler: polarWebhook,
});

export default http;
```

**Ingest Action Flow (`convex/ingest.ts`):**
1. Parse JSON body as `TelemetryPayload`
2. Validate API key (lookup in `apiKeys` table by hashed key)
3. Resolve `userId` from API key
4. Check user has active subscription (or is on free tier within limits)
5. Deduplicate events (check `processedEvents` table)
6. Fan out to appropriate mutations:
   - `snapshot` → upsert `instances` row, update sessions
   - `session_start` / `session_end` → upsert `sessions`
   - `llm_output` → insert `llmCalls`, update `dailyUsage`
   - `agent_end` → insert `agentRuns`
   - `tool_call` → insert `toolCalls`
   - `gateway_start` / `gateway_stop` → update `instances.status`
7. Mark event IDs as processed
8. Return `{ ok: true }` or `{ error: "..." }`

### 2.3 Convex Queries

**ClawHub Overview (`queries/clawhub.ts`):**
```typescript
// getClawHubOverview(userId)
// Returns:
{
  instances: { total: number; online: number; offline: number };
  today: {
    totalTokens: number;
    estimatedCost: number;
    llmCalls: number;
    activeSessions: number;
    agentRuns: number;
    errors: number;
  };
  thisMonth: {
    totalTokens: number;
    estimatedCost: number;
  };
  recentActivity: Array<{
    type: string;
    instanceName: string;
    agentId: string;
    description: string;
    timestamp: number;
  }>;
}
```

**Usage Analytics (`queries/usage.ts`):**
```typescript
// getUsageTimeSeries(userId, startDate, endDate, groupBy: "day" | "week" | "month")
// Returns daily/weekly/monthly token and cost data for charts

// getUsageByModel(userId, startDate, endDate)
// Returns breakdown by model (e.g., Claude Sonnet vs Opus vs Haiku)

// getUsageByInstance(userId, startDate, endDate)
// Returns breakdown by Openclaw instance

// getUsageByAgent(userId, startDate, endDate)
// Returns breakdown by agent ID
```

**Sessions (`queries/sessions.ts`):**
```typescript
// listSessions(userId, filters: { instanceId?, agentId?, status?, dateRange? }, pagination)
// Returns paginated session list with key metrics

// getSessionDetail(sessionId)
// Returns full session info with LLM calls, tool calls, agent runs
```

### 2.4 Convex Cron Jobs

```typescript
// convex/crons.ts
import { cronJobs } from "convex/_generated/server";

const crons = cronJobs();

// Mark instances as offline if no heartbeat in 5 minutes
crons.interval("check instance health", { minutes: 2 }, "mutations/instances:checkHealth");

// Clean up old deduplication records (older than 24h)
crons.daily("cleanup processed events", { hourUTC: 3 }, "actions/cleanup:pruneProcessedEvents");

export default crons;
```

### 2.5 Free Tier Limits

For users without a Polar subscription:

| Resource | Free Limit |
|----------|------------|
| Instances | 1 |
| Data retention | 7 days |
| LLM call history | Last 1,000 records |
| Snapshot frequency | Minimum 5 minutes |

Pro plan (via Polar): Unlimited instances, 90-day retention, full history, 60s sync.

---

## Part 3: Next.js Frontend

### 3.1 Project Structure

```
app/
├── (marketing)/
│   ├── page.tsx                    # Landing page
│   ├── pricing/page.tsx            # Pricing page (free vs pro)
│   └── layout.tsx                  # Marketing layout (no sidebar)
├── (clawhub)/
│   ├── layout.tsx                  # ClawHub layout (sidebar + topbar)
│   ├── clawhub/
│   │   └── page.tsx                # ClawHub overview
│   ├── instances/
│   │   ├── page.tsx                # All instances list
│   │   └── [instanceId]/
│   │       └── page.tsx            # Instance detail
│   ├── sessions/
│   │   ├── page.tsx                # Sessions list (filterable)
│   │   └── [sessionId]/
│   │       └── page.tsx            # Session detail (LLM calls, tools, timeline)
│   ├── usage/
│   │   └── page.tsx                # Usage analytics (charts, breakdowns)
│   ├── costs/
│   │   └── page.tsx                # Cost analytics (by model, agent, time)
│   ├── agents/
│   │   └── page.tsx                # Agent overview across instances
│   ├── settings/
│   │   ├── page.tsx                # General settings
│   │   ├── api-keys/page.tsx       # API key management
│   │   └── billing/page.tsx        # Polar subscription management
│   └── setup/
│       └── page.tsx                # Onboarding: install plugin, get API key
├── api/
│   └── webhooks/
│       └── clerk/route.ts          # Clerk webhook (user sync to Convex)
├── layout.tsx                      # Root layout
├── globals.css
└── providers.tsx                   # ConvexProvider + ClerkProvider wrapper
```

### 3.2 Key Pages

#### ClawHub Overview (`/clawhub`)

The main landing page after login. Shows at-glance health of the user's Openclaw setup.

**Sections:**
1. **Instance Status Cards** - Each connected Openclaw instance shown as a card with online/offline indicator, last seen time, number of active sessions
2. **Today's Stats Bar** - Horizontal stats: Total tokens, Estimated cost, Active sessions, Agent runs, Errors
3. **Cost Trend Chart** - Line chart showing daily cost for the last 30 days (recharts or similar)
4. **Recent Activity Feed** - Chronological list of recent events (session starts, agent completions, errors)
5. **Top Agents** - Which agents are most active today

#### Usage Analytics (`/usage`)

Deep-dive into token usage patterns.

**Sections:**
1. **Date Range Picker** - Select time window
2. **Usage Over Time** - Stacked area chart (input tokens, output tokens, cache tokens) by day
3. **Usage by Model** - Bar chart comparing token usage across models (Sonnet, Opus, Haiku, etc.)
4. **Usage by Agent** - Table showing per-agent token breakdown
5. **Usage by Instance** - For multi-instance users, compare across machines

#### Cost Analytics (`/costs`)

Focus on spend tracking and projections.

**Sections:**
1. **Monthly Spend Summary** - Current month total, comparison to last month, daily average
2. **Cost Over Time** - Line chart with cost per day
3. **Cost by Model** - Pie chart showing spend distribution across models
4. **Cost by Agent** - Table with per-agent cost breakdown
5. **Projected Monthly Cost** - Based on current daily average, project end-of-month spend

#### Sessions (`/sessions`)

Browse and filter all sessions across instances.

**Sections:**
1. **Filters** - By instance, agent, status (active/ended), date range, model
2. **Sessions Table** - Sortable table: Session ID, Agent, Instance, Started, Duration, Tokens, Cost, Status
3. **Session Detail** (click-through): Timeline view showing each LLM call, tool call, and event in order with token counts and latency

#### Instance Detail (`/instances/[instanceId]`)

Deep view into a specific Openclaw instance.

**Sections:**
1. **Health Status** - Online/offline, uptime, last seen
2. **Agents** - List of agents configured on this instance
3. **Channels** - Connected messaging channels and their status
4. **Active Sessions** - Currently running sessions
5. **Cron Jobs** - Scheduled tasks and their last run status
6. **Recent Activity** - Event feed for this instance

#### Setup/Onboarding (`/setup`)

Guided flow for new users to connect their first Openclaw instance.

**Steps:**
1. Generate an API key
2. Install the ClawHub plugin on their Openclaw (`openclaw plugin install openclaw-clawhub`)
3. Configure the plugin with API key
4. Verify connection (poll for first telemetry)

### 3.3 Component Library

Use **shadcn/ui** for consistent, accessible components:

```
components/
├── ui/                         # shadcn/ui primitives
│   ├── button.tsx
│   ├── card.tsx
│   ├── table.tsx
│   ├── badge.tsx
│   ├── dialog.tsx
│   ├── input.tsx
│   ├── select.tsx
│   ├── tabs.tsx
│   ├── chart.tsx               # recharts wrapper
│   └── ...
├── layout/
│   ├── sidebar.tsx             # Navigation sidebar
│   ├── topbar.tsx              # Top bar with user menu
│   └── mobile-nav.tsx          # Mobile navigation
├── clawhub/
│   ├── stats-bar.tsx           # Horizontal stat cards
│   ├── instance-card.tsx       # Instance status card
│   ├── activity-feed.tsx       # Recent activity list
│   ├── cost-chart.tsx          # Cost line chart
│   └── usage-chart.tsx         # Usage area chart
├── sessions/
│   ├── session-table.tsx       # Sessions data table
│   ├── session-timeline.tsx    # Session detail timeline
│   └── session-filters.tsx     # Filter controls
├── settings/
│   ├── api-key-manager.tsx     # Create/revoke API keys
│   └── billing-card.tsx        # Polar subscription status
└── shared/
    ├── date-range-picker.tsx   # Reusable date range selector
    ├── empty-state.tsx         # Empty state illustrations
    ├── loading-skeleton.tsx    # Loading states
    └── status-badge.tsx        # Online/offline/error badges
```

### 3.4 Charting

Use **Recharts** (widely used, works well with Next.js SSR):
- Line charts for cost/usage over time
- Stacked area charts for token breakdown
- Bar charts for model/agent comparisons
- Pie/donut charts for cost distribution

---

## Part 4: Clerk Authentication

### 4.1 Setup

- Clerk handles sign-up, sign-in, and user management
- Use Clerk's Next.js middleware for route protection
- Webhook syncs Clerk user creation/updates to Convex `users` table

### 4.2 Auth Flow

1. User signs up via Clerk (email/password or OAuth)
2. Clerk webhook fires → `api/webhooks/clerk/route.ts` → Convex mutation creates `users` row
3. On login, Clerk JWT is used for Convex auth via `ConvexProviderWithClerk`
4. All Convex queries/mutations validate user identity via `ctx.auth.getUserIdentity()`

### 4.3 Protected Routes

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";

const isProtectedRoute = createRouteMatcher(["/clawhub(.*)", "/instances(.*)", "/sessions(.*)", "/usage(.*)", "/costs(.*)", "/agents(.*)", "/settings(.*)", "/setup(.*)"]);

export default clerkMiddleware(async (auth, req) => {
  if (isProtectedRoute(req)) await auth.protect();
});
```

---

## Part 5: Polar Billing Integration

### 5.1 Plans

| | Free | Pro |
|---|---|---|
| Price | $0 | $X/month (TBD) |
| Instances | 1 | Unlimited |
| Data retention | 7 days | 90 days |
| Sync interval | 5 min minimum | 60s minimum |
| History | Last 1,000 LLM calls | Unlimited |
| Support | Community | Priority |

### 5.2 Integration Points

1. **Checkout**: From `/settings/billing` or upgrade prompts, redirect to Polar checkout with Clerk user ID as metadata
2. **Webhook**: `POST /webhooks/polar` Convex HTTP action handles subscription events:
   - `subscription.created` → set `users.plan = "pro"`, store `polarSubscriptionId`
   - `subscription.updated` → update plan status
   - `subscription.canceled` → set `users.plan = "free"` (at period end)
3. **Subscription Check**: Convex queries check `users.plan` to enforce limits
4. **Customer Portal**: Link to Polar customer portal for self-service billing management

### 5.3 Enforcement

Limits are enforced at two levels:
- **Ingest endpoint**: Rejects data if free user exceeds instance limit or sync frequency
- **UI queries**: Queries respect retention limits (filter by date based on plan)

---

## Part 6: Implementation Phases

### Phase 1: Foundation (Week 1-2)

**Goal:** Scaffold the project, get auth working, basic data flow.

1. **Initialize Next.js project** with TypeScript, Tailwind CSS, App Router
2. **Set up Convex** - install, configure, define initial schema
3. **Set up Clerk** - configure auth, add middleware, create webhook for user sync
4. **Set up shadcn/ui** - install base components
5. **Create layout** - sidebar navigation, topbar, responsive shell
6. **Build marketing landing page** - simple hero + feature highlights + CTA
7. **Build the Convex schema** - all tables and indexes as defined above
8. **Build the ingest HTTP action** - API key validation, event deduplication, basic mutation fan-out
9. **Build API key management** - Settings page to create/revoke keys

**Deliverable:** User can sign up, log in, generate an API key, and see an empty ClawHub.

### Phase 2: Openclaw Plugin (Week 2-3)

**Goal:** Build the plugin that pushes data from Openclaw to ClawHub.

1. **Scaffold plugin** - manifest, entry point, config schema
2. **Implement event collector** - hook into gateway lifecycle events
3. **Implement snapshot collector** - periodic full-state capture via gateway methods
4. **Implement HTTP pusher** - batch POST to Convex ingest endpoint with retry logic
5. **Implement deduplication** - event IDs, idempotent pushes
6. **Build the user-invocable skill** - `clawhub` skill for status checks
7. **Build the gateway method** - `clawhub.status` for plugin status
8. **Write installation docs** - step-by-step setup guide
9. **Test end-to-end** - install plugin locally, verify data arrives in Convex

**Deliverable:** Plugin installed on Openclaw pushes telemetry to Convex every 60s.

### Phase 3: ClawHub Core (Week 3-4)

**Goal:** Build the main ClawHub views with real data.

1. **ClawHub overview page** - stats bar, instance cards, activity feed
2. **Instance list page** - all instances with status
3. **Instance detail page** - health, agents, channels, sessions, cron
4. **Session list page** - filterable, sortable table with pagination
5. **Session detail page** - timeline view of LLM calls, tool calls, events
6. **Convex queries** - all queries for above pages
7. **Convex cron jobs** - instance health check, processed event cleanup
8. **Setup/onboarding page** - guided flow to connect first instance

**Deliverable:** Fully functional ClawHub showing real Openclaw data.

### Phase 4: Analytics & Costs (Week 4-5)

**Goal:** Rich usage analytics and cost tracking.

1. **Daily usage aggregation mutation** - update `dailyUsage` table on each ingest
2. **Usage analytics page** - time series charts, model breakdown, agent breakdown
3. **Cost analytics page** - spend tracking, projections, model cost comparison
4. **Date range picker** - reusable component for all analytics views
5. **Export functionality** - CSV download of usage/cost data

**Deliverable:** Full analytics suite with charts and cost tracking.

### Phase 5: Billing & Polish (Week 5-6)

**Goal:** Polar integration, free/pro enforcement, polish.

1. **Polar integration** - checkout flow, webhook handler, subscription management
2. **Billing settings page** - current plan, upgrade/downgrade, payment history
3. **Free tier enforcement** - limit instances, retention, sync frequency
4. **Upgrade prompts** - contextual prompts when hitting free tier limits
5. **Loading states** - skeletons for all pages
6. **Empty states** - illustrations and CTAs when no data
7. **Error handling** - error boundaries, retry logic, user-friendly messages
8. **Mobile responsiveness** - ensure all pages work on mobile
9. **Dark mode** - toggle between light/dark themes

**Deliverable:** Production-ready ClawHub with billing.

### Phase 6: Open Source & Launch (Week 6-7)

**Goal:** Prepare for open source release and hosted launch.

1. **Open source repo setup** - LICENSE (MIT or Apache 2.0), CONTRIBUTING.md, README with setup instructions
2. **Self-hosting guide** - instructions to deploy your own instance (Convex + Vercel + Clerk + Polar)
3. **Docker compose** (optional) - for self-hosted users who want a simple setup
4. **Environment variable documentation** - all required env vars with descriptions
5. **CI/CD** - GitHub Actions for lint, type-check, build
6. **Deploy hosted version** - Vercel + Convex production deployment
7. **Launch** - announce on Openclaw community channels

**Deliverable:** Open source repo live, hosted version deployed and accepting users.

---

## Part 7: Key Technical Decisions

### 7.1 Why Convex for this?

- **Real-time subscriptions** - ClawHub UI auto-updates when new telemetry arrives (no polling needed on frontend)
- **HTTP Actions** - Clean ingest endpoint for the plugin to POST to
- **Cron jobs** - Built-in scheduling for health checks and cleanup
- **TypeScript end-to-end** - Schema, queries, mutations, and frontend all type-safe
- **Scales well** - Managed infrastructure, no ops burden

### 7.2 Why a Plugin (not a Relay Agent)?

- **Native integration** - Hooks directly into Openclaw's event system, no external process
- **No extra process** - Runs inside the gateway, no separate daemon to manage
- **Rich data access** - Full access to gateway methods (sessions, usage, health, config)
- **User-controlled** - Standard Openclaw plugin install/config flow, familiar to users
- **Lightweight** - Just event hooks + periodic HTTP POST, minimal overhead

### 7.3 Deduplication Strategy

Events can be delivered more than once (network retries, plugin restarts). Handle via:
1. Each event gets a UUID (`eventId`)
2. Convex `processedEvents` table tracks seen IDs
3. Before processing, check if `eventId` exists → skip if so
4. Clean up old entries daily (Convex cron)

### 7.4 Cost Estimation

The plugin sends raw token counts. Cost estimation happens in Convex:
- Maintain a `modelPricing` lookup (hardcoded or configurable)
- On each `llm_output` event, compute cost from tokens × model price
- This keeps pricing logic centralized and updatable without plugin changes

### 7.5 Data Model for Future Teams Support

The schema is designed so adding teams later is straightforward:
- Add a `teams` table and `teamMembers` table
- Add optional `teamId` to `instances`, `apiKeys`
- Queries already filter by `userId` - extend to also accept `teamId`
- Minimal schema migration needed

---

## Part 8: Environment Variables

```bash
# Next.js / Vercel
NEXT_PUBLIC_CONVEX_URL=           # Convex deployment URL
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY= # Clerk frontend key
CLERK_SECRET_KEY=                  # Clerk backend key
CLERK_WEBHOOK_SECRET=              # Clerk webhook signing secret

# Convex
CLERK_JWT_ISSUER_DOMAIN=          # For Convex auth with Clerk

# Polar
POLAR_ACCESS_TOKEN=               # Polar API token
POLAR_WEBHOOK_SECRET=             # Polar webhook signing secret
POLAR_ORGANIZATION_ID=            # Your Polar org

# Plugin (set by user in their Openclaw config)
# CLAWHUB_API_KEY=                # Generated from ClawHub settings
# CLAWHUB_ENDPOINT=               # Defaults to hosted URL
```

---

## Summary

This plan delivers ClawHub, an open-source monitoring hub for Openclaw that:

1. **Connects natively** via an Openclaw plugin (no external processes)
2. **Syncs periodically** (60s default) with batched telemetry pushes
3. **Stores data** in Convex with real-time UI updates
4. **Authenticates** via Clerk with a simple API key linking the plugin to a user
5. **Monetizes** via Polar with a free tier and flat-rate pro subscription
6. **Shows everything** - instances, agents, sessions, usage, costs, channels, cron jobs
7. **Is open source** - anyone can self-host, or use the hosted version

Estimated build time: ~6-7 weeks for a solo developer, faster with a team.
