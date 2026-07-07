# AI System Reference — WeKraft SaaS (KAYA & HARRY)
> Last Updated: June 25, 2026

---

## Overview

WeKraft embeds two specialized AI agents directly into the project management workspace:

- **KAYA** — AI Product Manager: manages sprints, tasks, calendar events, delivery schedulers, and velocity reporting.
- **HARRY** — AI Senior Developer: reviews pull requests, analyzes codebases, identifies bugs, and provides architectural feedback.

Both agents use a LangGraph state machine pipeline with SSE streaming responses and a human-in-the-loop (HITL) interrupt mechanism for multi-step actions requiring user approval.

---

## 1. Agent Architecture

### 1.1 Request Flow

`
User (AiAssistantSheet) --> callAgentRoute() --> POST /api/agent --> SSE Proxy (proxy.ts) --> LangGraph Agent Server
                                                       |
                                                  Upstash Redis
                                                  (rate limit + NULL sentinel cache)
                                                       |
                                                  Convex (user lookup)
`

### 1.2 SSE Event Types

The agent server returns a Server-Sent Events stream. The client (useLangGraphAgent.ts) parses it and yields typed AgentEvent objects:

| SSE Event | Description |
|---|---|
| checkpoint | Graph state snapshot after a node execution |
| message_chunk | Streaming LLM token chunk (partial AI message) |
| interrupt | Human-in-the-loop pause — requires user decision |
| custom | Application-level status events (status, memory_loaded, error) |
| error | Error in graph node execution |

### 1.3 Agent State Schema

`	s
interface AgentState extends WithMessages {
  user_id?: string;     // Convex user ID
  user_name?: string;   // Display name for Mem0 memory context
  project_id?: string;  // Active project context
  thread_id: string;    // LangGraph thread ID for conversation continuity
}
`

### 1.4 Request Types

The agent supports four modes:

| Type | Purpose | Key Fields |
|---|---|---|
| run | New conversation or message | state: { messages, user_id, project_id, thread_id } |
| resume | Resume after HITL interrupt | resume: ResumeValue, thread_id |
| fork | Branch from a previous checkpoint | config: CheckpointConfig, state |
| replay | Re-run from a saved checkpoint | config: CheckpointConfig |

---

## 2. Human-in-the-Loop (HITL) Interrupt System

When KAYA plans a multi-step action (creating a sprint backlog, inserting a calendar event, setting up a scheduler), the LangGraph graph pauses and emits an interrupt event. The client renders an interactive approval card. The user reviews, optionally edits, and confirms or cancels.

### 2.1 Interrupt Payload Types

**CalendarEventInterrupt** (tool: "create_calendar_event")
`	s
{
  tool: "create_calendar_event",
  message: string,  // KAYA's explanation
  preview: {
    title: string,
    description: string,
    type: "event" | "milestone",
    start: string,  // ISO 8601
    end: string,    // ISO 8601
    allDay: boolean
  }
}
`
UI Component: CalendarApprovalCard.tsx

**SprintItemSelectionInterrupt** (tool: "add_items_to_sprint")
`	s
{
  tool: "add_items_to_sprint",
  sprint_id: string,
  message?: string
}
`
UI Component: SprintItemSelectionCard.tsx

**SchedulerSetupInterrupt** (tool: "setup_report_scheduler")
`	s
{
  tool: "setup_report_scheduler",
  message: string,
  existing_data?: {
    name: string,
    frequencyDays: number,
    recipientEmail?: string,
    isActive: boolean
  }
}
`
UI Component: SchedulerSetupCard.tsx

### 2.2 Resume Value Types

After the user interacts with the interrupt card, the frontend sends a resume request:

`	s
type ResumeValue =
  | { action: "cancel" }                                    // User cancelled
  | { action: "approve"; edits?: Partial<CalendarEvent> }   // Calendar event approved/edited
  | { task_ids: string[] }                                   // Sprint task selection confirmed
  | { name: string; frequencyDays: number; recipientEmail?: string; isActive: boolean }  // Scheduler config
`

### 2.3 Custom Status Events

During execution, KAYA emits custom events the UI can display as status indicators:

`	s
type KayaCustomEvent =
  | { type: "status"; message: string }         // "Searching your backlog..."
  | { type: "memory_loaded"; count: number; memories?: string[] }  // Mem0 context loaded
  | { type: "error"; message: string }          // Tool call failure
`

---

## 3. Agent API Route (/api/agent)

File: src/app/api/agent/route.ts

### 3.1 Security Layers

1. Rate Limiting (first check, fastest):
   - Upstash Redis sliding window: 3 requests per minute.
   - Identifier: userId (authenticated) OR IP address (x-real-ip or x-forwarded-for first segment) for anonymous.
   - Returns 429 with Retry-After header on limit exceeded.

2. NULL Sentinel Cache (ghost request protection):
   - User lookup keyed as "user:{userId}" in Upstash Redis.
   - Cached for 60 seconds.
   - Non-existent users cached as "__NULL__" string — prevents repeated Convex queries on probe attacks.
   - Redis errors are caught and silently bypassed (degraded mode: query Convex directly).

3. Pro Plan Gate:
   - Only users with accountType === "pro" can invoke KAYA.
   - Missing userId or null user resolves to null (not pro) — no fallback bypass.
   - Check runs in parallel with the upstream AI request (non-blocking).

### 3.2 SSE Proxy Pattern

`
1. Start both the AI request (fetch to NEXT_PUBLIC_AGENT_URL/agent) and user check in parallel.
2. Immediately create a TransformStream and return the readable end as the HTTP response.
3. In a background async IIFE, await both promises.
4. If user fails the pro check, write an SSE error event and close the stream.
5. If user passes, pipe the upstream SSE response body directly into the transform stream.
`

This ensures the HTTP response headers (Content-Type: text/event-stream) are sent immediately, before any async work completes, preventing client-side connection timeouts.

### 3.3 Request Body Schema (sent to NEXT_PUBLIC_AGENT_URL/agent)

`json
{
  "type": "run",
  "thread_id": "session-abc-123",
  "user_id": "j57e3b487ag3xrbdgwc0c4hpm987kcq4",
  "user_name": "Alice",
  "project_id": "k1234projid",
  "state": {
    "messages": [
      {
        "id": "msg-1",
        "role": "user",
        "parts": [
          {
            "type": "text",
            "text": "Plan a sprint for the next 2 weeks"
          }
        ]
      }
    ]
  }
}
`

For resume after HITL interrupt:
`json
{
  "type": "resume",
  "thread_id": "session-abc-123",
  "user_id": "j57e3b487ag3xrbdgwc0c4hpm987kcq4",
  "resume": { "action": "approve" }
}
`

---

## 4. Vercel AI SDK Integration (Chatbot)

Separate from the LangGraph agent, WeKraft also uses the Vercel AI SDK for simpler chatbot interactions (src/modules/chatbot, /api/chatbot).

Chatbot endpoint uses:
- Rate limited by chatbotRatelimit (3 req/min per user) and chatbotGlobalRatelimit (10 req/min global).
- Vercel AI SDK streaming response format (useChat hook compatible).

---

## 5. KAYA Teamspace Integration (/api/kaya-teamspace)

KAYA can participate in Teamspace chat channels:
- Endpoint: /api/kaya-teamspace
- Triggered by @KAYA mentions in channel messages.
- Same LangGraph pipeline and SSE streaming, but responses are posted back into the Turso channel as messages.
- Controlled per-project by the canUseAITeamspace flag in projectDetails.

---

## 6. Convex Agent Tools (internalMutation / internalQuery)

File: convex/agentTools.ts

These are Convex server-side functions the LangGraph agent can call via tool use. They are all internal — cannot be called from the browser client.

| Function | Type | Purpose |
|---|---|---|
| insertCalendarEvent | internalMutation | Create a calendar event for a project |
| getSprintPlannerContext | internalQuery | Get project deadline, sprint names, unassigned task count, days-to-deadline |

The agent calls these via the LangGraph tool-calling mechanism. On success, the graph continues. On failure, a custom error event is emitted.

---

## 7. Memory System (Mem0)

KAYA uses Mem0 Cloud for persistent semantic memory across sessions:

- **Short-term memory (InMemorySaver)**: Within a single thread_id session — conversation history is preserved.
- **Long-term memory (Mem0)**: Across sessions — KAYA recalls past sprint decisions, tech stack context, team dynamics, and user preferences.

On session start, KAYA loads relevant memories from Mem0 and emits a custom event:
`json
{ "type": "memory_loaded", "count": 5, "memories": ["User is building a B2B SaaS...", "..."] }
`

---

## 8. HARRY Agent

File: src/modules/ai/HarryAssistantSheet.tsx

HARRY is the AI Senior Developer agent, designed for codebase analysis:
- Operates within the HeatmapFlow / codebase visualization interface.
- Reviews pull request diffs via GitHub API.
- Analyzes repository structure and commit history.
- Suggests bug fixes and architectural improvements.
- Uses Anthropic Claude 3.5 Sonnet (configured on the agent server).

HARRY shares the same /api/agent SSE proxy and LangGraph infrastructure as KAYA, but is a separate graph with different system prompt, tools, and model configuration.

---

## 9. Rate Limiting Summary

File: src/lib/rate-limit.ts

| Limiter | Prefix | Window | Limit | Used By |
|---|---|---|---|---|
| ratelimit | agent_rl | 1 minute | 3 req | /api/agent (KAYA) |
| chatbotRatelimit | chatbot_rl | 1 minute | 3 req | /api/chatbot (per user) |
| chatbotGlobalRatelimit | chatbot_global_rl | 1 minute | 10 req | /api/chatbot (global) |

All limiters use Upstash Redis sliding window algorithm.
