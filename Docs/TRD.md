# Technical Requirements Document (TRD) — Wekraft SaaS
**System Architecture, Technology Stack, and Technical Debt Audit**
> Last Updated: June 25, 2026 — Full codebase audit of v0.1.0

---

## 1. Executive Summary & Project Overview

**Wekraft SaaS** is a high-performance, real-time collaboration and project management platform designed for developers and product teams. It integrates traditional project management workflows (Sprints, Kanban boards, Task tracking, and Issue management) with a unified workspace containing:
- **Real-Time Teamspace**: Chat channels, threaded conversations, poll voting, link unfurling, collaborative documents, and full-text search.
- **Agentic AI Assistance ("KAYA" & "HARRY")**: Two persistent AI agents with LangGraph pipeline, semantic memory via Mem0, and human-in-the-loop interrupt card UI.
- **Location-Based Payment Routing**: Dual-payment paths dynamically routing domestic Indian transactions via Razorpay and global cards via Stripe (with LemonSqueezy as an additional provider).
- **GitHub-Native Integration**: Repository sync, issue import, PR analysis, webhook event routing, and codebase heatmap visualization.
- **Edge-Optimized Collaborative Features**: Integrated video meetings (Stream Video SDK), project scheduling, automated delivery alerts (Inngest), analytics dashboards, and time intelligence (Project Timeline, Delay Debt Analyzer, Pace Tracker).
- **IDE Extension**: VS Code extension with OAuth handshake (5-min tokens) for two-way task sync.
- **Customer Desk**: Lightweight CRM for per-project customer management and service request triage.
- **Admin Panel**: Internal platform administration dashboard gated by isAdmin flag.

---

## 2. Core Technology Stack

Wekraft operates on a highly decentralized, serverless architecture that leverages specialized edge services for specific workloads.

`
graph TD
    Client[Next.js Client v16.1.6 / React 19] -->|Auth Token| Clerk[Clerk Auth]
    Client -->|GraphQL Queries / Mutations| Convex[Convex DB Serverless v1.32.0]
    Client -->|SQL / HTTP API| Turso[Turso Database LibSQL]
    Client -->|SSE Channel Subscriptions| Ably[Ably Pub/Sub Router]
    Client -->|Video & Audio Streams| Stream[GetStream Video SDK]
    Client -->|Checkout Redirect| Stripe[Stripe Checkout]
    Client -->|Modal SDK Checkout| Razorpay[Razorpay Subscription SDK]
    Client -->|ReactFlow Canvas| Heatmap[Codebase Heatmap Viz]

    subgraph Serverless Backend (Next.js API Routes)
        Proxy[/api/agent] -->|IP / UserID Rate Limit| Upstash[Upstash Redis + RateLimit]
        Proxy -->|SSE Stream Proxy| ExternalAgent[LangGraph AI Agent Engine]
        KayaTeamspace[/api/kaya-teamspace] --> Upstash
        KayaTeamspace --> ExternalAgent
        Paymentsstripe[/api/payments/stripe] --> StripeAPI[Stripe Node SDK]
        Paymentsrazorpay[/api/payments/razorpay] --> RazorpayAPI[Razorpay SDK]
        PaymentsLemon[/api/payments/lemonsqueezy] --> LemonAPI[Lemon Squeezy API]
    end

    Convex -->|Triggers Alert Workflows| Inngest[Inngest Event Engine]
    Inngest -->|Fires Notifications| Convex
    Convex -->|GitHub App Webhooks| GitHub[GitHub / Octokit]
    Convex -->|Extension OAuth API| ExtAPI[extensionApi.ts]
`

### Stack Breakdown by Layer

| Layer | Technologies Used | Purpose & Implementation |
| :--- | :--- | :--- |
| **Frontend Framework** | Next.js 16.1.6 (Canary), React 19.2.3, TypeScript | Core SPA with app-router structure, server actions, and Serverless route support. |
| **Styling & UX** | Tailwind CSS 4, Radix UI, Base UI, Framer Motion, GSAP, Lenis | Rich, hardware-accelerated animations, smooth scroll, layout transitions, and premium dark/light mode interfaces. |
| **Realtime Gateway** | Ably (bly v2.21.0) | Pub/Sub event bus handling channel chat messages, presence events, and live team meeting synchronization. |
| **Operational DB** | Convex (convex v1.32.0) | Relational application database (users, tasks, sprints, calendar events, notifications, meetings, customer desk, agent tools). |
| **Chat DB (Edge)** | Turso / libSQL (@libsql/client v0.17.2) | SQLite-compatible database distributed to the edge. Stores high-throughput messages, reactions, channels, settings, and full-text index (fts5). |
| **Cache & Limits** | Upstash Redis (@upstash/redis & @upstash/ratelimit) | Low-latency state caching for user objects, and sliding-window rate limiting on API routes (per-IP and per-API-key). |
| **Auth & Identity** | Clerk (@clerk/nextjs v7.0.4) | Secure authentication, OAuth providers (GitHub), session tokens, and middleware protection. |
| **Async Workflows** | Inngest (inngest v4.1.0) | Event-driven serverless cron jobs and background tasks (project duration alerts, scheduler runs). |
| **Payments** | Stripe + Razorpay (azorpay v2.9.5) + LemonSqueezy | Tri-gateway billing. Stripe: international cards. Razorpay: Indian UPI/cards. LemonSqueezy: alternate global provider. |
| **Video Meeting** | Stream Video SDK (@stream-io/video-react-sdk v1.37.4) | Interactive multi-party video calling embedded in team meets. |
| **Agentic AI** | Vercel AI SDK (i v6), LangChain (@langchain/core, @langchain/langgraph v1.2.8), @langchain/langgraph-sdk | Streaming AI backend powered by LangGraph state machines, LangGraph SDK, and Mem0 (semantic memory). |
| **AI Models** | OpenAI GPT-4o-mini / GPT-4.1-nano (KAYA), Anthropic Claude 3.5 Sonnet (HARRY) | Dual-model strategy: cheap routing for PM tasks, powerful model for code reviews. |
| **GitHub Integration** | Octokit (octokit v5.0.5), GitHub App Webhooks | Repository sync, issue import, PR analysis, commit history, webhook event routing. |
| **Codebase Visualization** | @xyflow/react v12.10.2 | ReactFlow-based heatmap canvas for codebase contribution visualization. |
| **Calendar** | @fullcalendar/react v6.1.20 (daygrid, timegrid, interaction) | Full-featured project calendar with event and milestone types. |
| **Drag & Drop** | @dnd-kit/core, @dnd-kit/sortable, @dnd-kit/utilities | Kanban board drag-and-drop task/issue management. |
| **Charts** | echarts v2.15.4 | Sprint analytics, velocity charts, radar charts, workload distribution. |
| **Markdown** | @uiw/react-md-editor, eact-markdown, emark-gfm | Rich text rendering in task descriptions, channel messages, and KAYA responses. |
| **File Storage** | AWS S3 (@aws-sdk/client-s3) | Task and issue attachments, channel file uploads. |
| **Email** | Resend (esend v6.12.4), Nodemailer | Delivery scheduler notifications, transactional emails. |
| **Error Monitoring** | Sentry (@sentry/nextjs v10.56.0) | Client + server + edge error tracking with separate config files. |
| **Analytics** | Vercel Analytics (@vercel/analytics) | Page-level analytics on Vercel infrastructure. |
| **Onboarding Tours** | Driver.js (driver.js v1.4.0) | One-time feature tutorial overlays per feature area. |
| **Document Export** | docx v9.6.1 | Export project documents as Word files. |
| **State Management** | Zustand (zustand v5.0.12) | Client-side global store for UI state. |
| **Forms** | React Hook Form (eact-hook-form v7) + Zod (zod v4.3.6) | Validated form handling across the application. |

---

## 3. Storage Architecture (Hybrid Approach)

Wekraft utilizes a hybrid storage model to optimize performance:

1. **Primary Operational Storage (Convex)**: Serves as the transactional store for all application state.
   - Tables: users, userDetails, projects, projectMembers, projectJoinRequests, projectDetails, 	asks, 	askComments, 	askAssignees, issues, issueComments, issueAssignees, sprints, epositories, calendarEvents, schedulers, 	eam_meets, 
otifications, 	ickets, projectUpvoteRecords, serviceCustomers, serviceRequests, supportQueries, userApiKeys, piKeyRateLimits, handshakeTokens

2. **High-Throughput Chat Messaging (Turso)**: Edge-distributed database dedicated to absorbing chat traffic.
   - Stores: messages, reactions, channels, channel settings, full-text search index (fts5), member presence.

3. **File/Attachment Storage (AWS S3)**: Binary object storage for task/issue attachments and channel file uploads.

4. **Client-Side Cache (IndexedDB)**: Soft-expiry pattern (55-minute soft TTL, 24-hour hard TTL) for Turso message data. Returns stale-while-revalidate and spins background refresh.

5. **Low-Latency Cache (Upstash Redis)**: User object caching for agent route lookups. NULL sentinel caching (__NULL__) to prevent cache penetration on ghost requests.

---

## 4. Key Architectural Integrations

### 4.1. Dynamic Payment Routing Engine

Located in src/modules/web/Pricing.tsx:
- On pricing page initialization, the client queries IP geolocation (ipapi.co/json/).
- If the country is **India (IN)**, Rupees (Rs.) are rendered, routing flows through the useRazorpay hook.
- If the country is **Non-India**, USD ($) is rendered, redirecting to a Stripe hosted Checkout Session.
- Webhooks for Stripe (checkout.session.completed, customer.subscription.updated, customer.subscription.deleted) and Razorpay (subscription.charged, subscription.cancelled, subscription.halted, subscription.paused, subscription.resumed) securely invoke Convex database mutations to update user account tiers.
- **LemonSqueezy**: Additional payment provider with convex/lemonsqueezy.ts mutations (updatePlanServerSideInternal as internalMutation).
- A fallback auto-downgrade cron runs daily at **00:30 UTC** in Convex to process expired plans if webhooks fail.

### 4.2. Stream Video Call Engine

Located in convex/notifications.ts and src/modules/team-meet:
- Starting a call inserts a 	eam_meets record. A notification is fanned out to all project members.
- The notification body carries the Stream call ID, which deep-links to /workspace/meet/[meetingId].
- Participant joins are appended to the meeting document.
- Stale meetings (where hosts closed their tab without ending the call) are cleaned up by a daily Convex cron job running at **03:00 UTC** (marking active meetings older than 4 hours as inactive).

### 4.3. LangGraph AI Agent Architecture

The KAYA and HARRY agents are deployed as LangGraph state machines:
- **Frontend**: AiAssistantSheet.tsx (KAYA), HarryAssistantSheet.tsx (HARRY) send requests to /api/agent.
- **Transport**: SSE streaming via src/proxy.ts which proxies to the external LangGraph agent endpoint.
- **Rate Limiting**: Upstash Redis sliding window on /api/agent — IP-based fallback (x-real-ip/x-forwarded-for hashing) prevents anonymous key starvation.
- **NULL Sentinel Cache**: Non-existent user lookups are cached in Redis as __NULL__ for 60 seconds to block ghost request attacks.
- **Memory**: Mem0 cloud provides persistent semantic memory per user across sessions.
- **Tools**: convex/agentTools.ts exposes internalMutation and internalQuery functions (e.g., insertCalendarEvent, getSprintPlannerContext) that agents call server-side.
- **Human-in-the-Loop**: Agent returns structured interrupt payloads that the client renders as SprintItemSelectionCard, CalendarApprovalCard, and SchedulerSetupCard components. User confirms before mutations are committed.

### 4.4. IDE Extension OAuth Handshake

Located in convex/extensionApi.ts and src/app/extension:
1. Extension opens http://[host]/extension?callback_url=vscode://wekraft.wekraft/auth
2. User clicks "Grant Access to IDE" → createHandshakeToken() inserts a 5-minute token into handshakeTokens table.
3. Browser redirects to scode://wekraft.wekraft/auth?token=<hex>.
4. Extension calls exchangeHandshakeToken({ token }) via Convex HTTP endpoint.
5. Returns { userId, apiKey } — token is deleted immediately (one-time use).
- API keys have wk_ prefix + 64 hex chars. Stored in userApiKeys table.
- Per-key sliding window rate limiting via piKeyRateLimits table (one document per window start time).

### 4.5. Inngest Event-Driven Scheduling

Located in src/inngest and convex/scheduler.ts / convex/scheduleRunner.ts:
- Project delivery schedulers are configured per-project with requencyDays (minimum 3) and ecipientEmail.
- Inngest fires events based on 
extRunAt timestamps.
- KAYA can activate schedulers via tool call. Status tracked as isActive, isRunning, lastRunAt, lastRunStatus.

### 4.6. GitHub Integration Architecture

Located in convex/repo.ts, src/modules/github, src/modules/repo:
- GitHub App webhook handler processes push, pull_request, issues events.
- Issues are imported into the WeKraft issues table with 	ype: "github" and githubIssueUrl.
- epositories table stores githubId, epoName, epoOwner, epoFullName, isWebhookConnected, language.
- Codebase heatmap (HeatmapPanel.tsx) uses @xyflow/react to visualize contribution patterns.

---

## 5. Scheduled Cron Jobs

| Job Name | Schedule | Handler | Purpose |
| :--- | :--- | :--- | :--- |
| cleanup-old-notifications | Daily at 02:00 UTC | 
otifications.deleteOldNotifications | Delete notifications older than 30 days |
| downgrade-expired-plans | Daily at 00:30 UTC | payments.downgradeExpiredPlans | Safety-net for missed subscription.deleted webhooks |
| cleanup-stale-meetings | Daily at 03:00 UTC | 
otifications.markStaleMeetingsInactive | Mark team meets older than 4 hours as inactive |

---

## 6. Technical Debt & Production Vulnerabilities Audit

The codebase was audited for performance bottlenecks and system design failure modes. While major issues have been addressed, several critical optimizations remain.

### 6.1. System Design Failure Matrix

#### Cache Penetration (Ghost Request Attacks)
- **Status:** Partially Resolved
- **Fixes Applied:**
  - /api/agent/route.ts caches non-existent user lookups (__NULL__ sentinels) in Upstash Redis for 60 seconds.
  - The fallback treating unauthenticated requests as Pro users is deleted.
- **Remaining Debt:**
  - getUserByClerkToken scan (High Severity): In convex/user.ts, if the exact token lookup fails, the query loads every user into memory using ctx.db.query("users").collect() to perform suffix matching.
  - Proposed Fix: Force unified provider|id format on insert and query exclusively through the y_token index. Remove the .collect() scan entirely.

#### Thundering Herd (Cache Stampede)
- **Status:** Partially Resolved
- **Fixes Applied:**
  - Client-side IndexedDB chat caching (src/lib/db.ts) uses a Soft Expiry Pattern (CACHE_SOFT_TTL_MS of 55 minutes).
  - initTeamspaceDB() cold start query stampede (src/lib/turso/schema.ts) queries sqlite_master once to check initialization state.
- **Remaining Debt:**
  - deleteOldNotifications scan (High Severity): In convex/notifications.ts, the daily cleanup cron filters notifications using .filter() — a full database scan not backed by a Convex index.
  - Proposed Fix: Create a Convex index on createdAt and use .withIndex().
  - No Channel List Caching (Medium Severity): In src/app/api/teamspace/channels/route.ts, every channel render runs a 3x subquery SELECT against Turso.
  - Proposed Fix: 30-second Redis cache per user-project, invalidated on new channel insertion.

#### Hot Keys (Shard Overloads)
- **Status:** Partially Resolved
- **Fixes Applied:**
  - /api/agent/route.ts rate limiting hashes the client's IP (x-real-ip/x-forwarded-for) as a fallback rather than a shared "anonymous" string.
  - Chat message submissions use INSERT OR IGNORE to deduplicate retries on clientId.
- **Remaining Debt:**
  - getUniqueTags scan (High Severity): In convex/workspace.ts, when fetching tags for a project, the query collects all tasks into memory. This is an O(n) query that fires reactively on every task update.
  - Proposed Fix: Denormalize project tags into their own projectTags table. The read query becomes O(1).

---

## 7. Technical Debt Summary Table

| Severity | Component / Path | Technical Debt Description | Proposed Remediation |
| :--- | :--- | :--- | :--- |
| HIGH | convex/user.ts | Suffix-matching fallback loads all users into memory (users.collect()). | Remove suffix matching fallback; enforce strict Clerk format indexed on y_token. |
| HIGH | convex/notifications.ts | Notification cleanup cron scans entire DB table using unindexed .filter(). | Add .index("by_created_at", ["createdAt"]) and query via .withIndex(). |
| HIGH | convex/workspace.ts | getUniqueTags reactive query loads all project tasks to compute tags. | Denormalize tags into a project metadata table updated via mutators. |
| MEDIUM | channels/route.ts | Uncached channel list query runs 3 SQL subqueries on every navigation. | Cache the channel list in Upstash Redis for 30s; invalidate on channel updates. |
| MEDIUM | src/modules/web/Pricing.tsx | Pricing page relies on free, un-fallback-guarded IP API (ipapi.co). | Move geolocation parsing to Vercel edge headers (x-vercel-ip-country) or configure a fallback provider. |
| LOW | package.json | livekit-client and livekit-server-sdk are installed but not actively used (Stream Video SDK is the active choice). | Audit dependency list and remove unused LiveKit packages to optimize bundle size. |

---

## 8. API Route Inventory

| Route | Method | Auth | Purpose |
| :--- | :--- | :--- | :--- |
| /api/agent | POST | Clerk + Upstash rate limit | SSE proxy to LangGraph KAYA agent |
| /api/kaya-teamspace | POST | Clerk + rate limit | KAYA participation in teamspace channels |
| /api/payments/stripe/checkout | POST | None (rate limit recommended) | Generate Stripe Checkout Session |
| /api/payments/stripe/webhook | POST | Stripe signature | Handle Stripe subscription events |
| /api/payments/stripe/portal | POST | Clerk + ownership check | Generate Stripe Billing Portal URL |
| /api/payments/razorpay/subscription | POST | None (rate limit recommended) | Create Razorpay subscription order |
| /api/payments/razorpay/verify | POST | HMAC signature | Verify payment, fulfill plan |
| /api/payments/razorpay/cancel | POST | Clerk + ownership check | Cancel Razorpay subscription at period end |
| /api/payments/razorpay/webhook | POST | Razorpay HMAC | Handle Razorpay subscription events |
| /api/teamspace/channels | GET | Clerk | Fetch project channel list |
| /api/teamspace/[channelId]/messages | GET/POST | Clerk | Read/write Turso messages |
| /api/ai | POST | Clerk | Direct AI completion endpoint |
| /api/chatbot | POST | Clerk | Chatbot Node interaction |
| /api/attachments | POST | Clerk | S3 file upload handler |
| /api/issue-attachments | POST | Clerk | S3 issue file upload handler |
| /api/admin | POST | Clerk + isAdmin | Admin panel operations |
| /api/analytics | GET | Clerk | Platform analytics queries |
| /api/invite | GET | None | Public invite link resolver |
| /api/inngest | POST | Inngest signature | Inngest event handler endpoint |
| /api/webhooks | POST | Signature | GitHub/external webhook receiver |
| /api/public | GET | None | Public-facing project data |
| /api/objects | GET/POST | Clerk | S3 object management |
| /api/dashboard | GET | Clerk | Dashboard aggregation queries |
| /api/contact | POST | None | Public contact form handler |

---

## 9. Convex Backend Module Inventory

| File | Responsibility |
| :--- | :--- |
| schema.ts | Full database schema definition (25+ tables) |
| user.ts | User CRUD, subscription state, onboarding, skills, admin flags |
| project.ts | Project lifecycle, membership, join requests, upvotes, public discovery |
| workspace.ts | Task management, tag queries, filtering, kanban operations |
| sprint.ts | Sprint CRUD, task/issue assignment to sprints, stats |
| issue.ts | Issue tracker CRUD, GitHub issue import, severity/environment management |
| 
otifications.ts | Real-time notifications, stale meeting cleanup, delivery |
| gentTools.ts | Internal tool functions for KAYA/HARRY agents (insert events, query context) |
| extensionApi.ts | VS Code extension OAuth handshake, API key management, rate limiting |
| 	eamspaceAgents.ts | Teamspace AI agent message routing |
| calendar.ts | Calendar event CRUD |
| scheduler.ts | Delivery scheduler configuration and management |
| scheduleRunner.ts | Inngest-triggered scheduler execution logic |
| payments.ts | Auto-downgrade cron (downgradeExpiredPlans) |
| azorpay.ts | Razorpay DB mutations (updatePlan, verifySubscriptionOwner) |
| lemonsqueezy.ts | LemonSqueezy DB mutations (updatePlanServerSideInternal) |
| pricing.ts | PLAN_CONFIGS — plan limits and feature gates per tier |
| http.ts | Convex HTTP API routes (stripe.ts webhook handlers inlined) |
| epo.ts | GitHub repository sync and webhook processing |
| customerDesk.ts | Customer Desk CRUD (serviceCustomers, serviceRequests) |
| dmin.ts | Admin panel queries and mutations |
| piKeys.ts | API key management helpers |
| nalytics.ts | Analytics data queries |
| crons.ts | Cron job definitions (3 scheduled jobs) |
| uth.config.ts | Clerk auth configuration for Convex |
| projectDetails.ts | Project configuration (memberCanCreate, memberUseKaya, alerts) |
| support.ts | Support query submission |

---

## 10. Production-Grade Practice Rating

### Overall Rating: 8.3 / 10

#### Strengths (Why it rates highly):
- **Sophisticated AI Architecture:** LangGraph state machines with human-in-the-loop interrupt cards and Mem0 persistent memory is a genuinely advanced agentic implementation — not just a chatbot wrapper.
- **Modern Infrastructure Choice:** Convex (reactive state), Turso (edge SQL), Ably (SSE networking), Inngest (event workflows) form a cohesive, low-latency stack optimized for real-time collaboration.
- **Robust Security Hygiene:** Clerk is used comprehensively, routes check ownership of resource tokens before actions (subscription cancels, portal redirects), webhook signatures use crypto.timingSafeEqual, and upgradeAccount is protected as an internalMutation.
- **Edge Caching Patterns:** Soft-expiry IndexedDB architecture with background refresh handles connection loss and prevents client-side rendering bottlenecks cleanly.
- **Tri-Gateway Payments:** Rare dual-payment routing (Razorpay + Stripe + LemonSqueezy) with graceful cancellation, cron safety-net, and timing-safe HMAC verification.
- **Comprehensive Monitoring:** Sentry on all three runtime targets (client, server, edge), plus Vercel Analytics.

#### Areas for Improvement (Why it lost points):
- **Convex Query Architecture:** Unindexed collection scans (.collect()) in user.ts, 
otifications.ts, and workspace.ts are production timebombs when record counts grow.
- **SaaS Geolocation Fragility:** Free tier dependency on ipapi.co can cause Indian users to hit Stripe (with card decline risk) or global users to hit Indian pricing.
- **Unused Dependencies:** livekit-client and livekit-server-sdk increase bundle size with no current activation path.
