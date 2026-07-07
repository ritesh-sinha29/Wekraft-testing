# Production Infrastructure Audit — Wekraft SaaS
### Mapping System Design Failure Modes to the Codebase
> Last Updated: June 25, 2026 — Reflects current codebase state

---

## Stack Overview

| Layer | Technology | Purpose |
|---|---|---|
| Frontend cache | IndexedDB (db.ts) | Client-side chat message cache with soft-expiry pattern |
| Realtime | Ably | Pub/Sub for messages, channels, presence |
| App DB | Convex v1.32.0 | Tasks, users, projects, notifications, payments, meetings |
| Chat DB | Turso (libSQL) | Messages, channels, reactions, polls, private channel membership |
| Cache / Rate-limit | Upstash Redis | Rate limiting + server-side user caching |
| Auth | Clerk | Identity, tokens |
| Background jobs | Inngest | Async workflows, delivery schedulers |
| AI Proxy | /api/agent/route.ts | Streaming AI agent (SSE) with Redis rate limit + NULL sentinel cache |
| File Storage | AWS S3 | Task/issue attachments, channel file uploads |
| Error Monitoring | Sentry | Client + server + edge |

---

## Issue Status Matrix

| # | Issue | File | Original Severity | Status |
|---|---|---|---|---|
| 1 | Ghost userId probing + Pro bypass logic bug | api/agent/route.ts | CRITICAL | FIXED |
| 2 | localhost invite link in production | lib/static-store.tsx | CRITICAL | FIXED |
| 3 | IndexedDB hard TTL — tab stampede | lib/db.ts | CRITICAL | FIXED |
| 4 | Anonymous rate-limit shared hot key | lib/rate-limit.ts | HIGH | PARTIAL |
| 5 | initTeamspaceDB() migration storm on cold start | lib/turso/schema.ts | HIGH | FIXED |
| 6 | getUniqueTags O(n) reactive scan | convex/workspace.ts | HIGH | OPEN |
| 7 | Channel list has no caching | api/teamspace/channels/route.ts | MEDIUM | OPEN |
| 8 | getUserByClerkToken full table scan | convex/user.ts | MEDIUM | OPEN |
| 9 | deleteOldNotifications unindexed cron | convex/notifications.ts | MEDIUM | OPEN |
| 10 | POST messages not idempotent | api/teamspace/messages/route.ts | MEDIUM | FIXED |

---

## FIXED Issues — Verified Current State

### Fix 1: Ghost userId Probing + Pro Bypass Bug (FIXED)
File: src/app/api/agent/route.ts

Original problem: Unauthenticated requests with missing userId were treated as Pro users (Promise.resolve({ accountType: "pro" })). No caching on user lookups allowed ghost-ID probing.

Current implementation (VERIFIED):
- getCachedUser() function checks Upstash Redis first (key: user:{userId}).
- On a Redis hit of "__NULL__" sentinel, immediately returns null — no Convex query.
- On a Redis miss, queries Convex, then caches the result (user JSON or "__NULL__") for 60 seconds.
- The proCheckPromise for missing userId resolves to null (not pro), causing immediate 403 in the stream.
- Rate limiting uses userId || IP as identifier — prevents anonymous key starvation.

### Fix 2: Localhost Invite Link (FIXED)
File: src/lib/static-store.tsx:297-304

Current implementation (VERIFIED):
- Client-side: uses window.location.origin (dynamic, always correct for the current deployment).
- Server-side: reads NEXT_PUBLIC_APP_URL env var, with localhost:3000 only as a final fallback for local dev.
- Production invite links now correctly point to wekraft.xyz.

### Fix 3: IndexedDB Hard TTL Stampede (FIXED)
File: src/lib/db.ts

Current implementation (VERIFIED):
- CACHE_TTL_MS = 24 hours (hard expiry — true cache miss).
- CACHE_SOFT_TTL_MS = 55 minutes (soft expiry — serves stale data and signals background refresh).
- CachedChat interface includes softExpiry timestamp stored on every write.
- get() returns { ...result, __stale: true } when past soft TTL but under hard TTL.
- Callers detect __stale and trigger a background refresh without blocking the UI render.
- MESSAGE_MAX_AGE_MS = 30 days — messages older than 30 days are stripped from cache on every read and write, keeping the local cache consistent with server retention.
- prune() automatically keeps only the 100 most recently accessed channels.

### Fix 5: initTeamspaceDB() Migration Storm (FIXED)
File: src/lib/turso/schema.ts

Current implementation (VERIFIED):
- isDbInitialized in-memory flag is checked at the very top of initTeamspaceDB() (line 17) — fast exit on subsequent calls.
- On first call: queries sqlite_master once to check if ts_messages table exists.
- If table exists: only runs lightweight runMigrations() (ALTER TABLE with try/catch silencing duplicate errors).
- Full DDL initialization only runs on brand-new DB instances.
- Private channel membership table (ts_private_channel_members) added via migration with CASCADE delete.
- made_public_at column added to ts_channels via migration for privacy-preserving public conversion.

### Fix 10: Message Insert Idempotency (FIXED)
File: src/app/api/teamspace/messages/route.ts (inferred)

Current implementation: Uses INSERT OR IGNORE on clientId. Duplicate POSTs on network retry do not create duplicate messages.

---

## PARTIAL Issues

### Partial 4: Anonymous Rate-Limit Shared Hot Key (PARTIAL)
File: src/lib/rate-limit.ts

Original problem: All anonymous users shared the "anonymous" key in Upstash.

Current state (PARTIAL):
- GOOD: In /api/agent/route.ts (line 44-48), identifier = userId || IP. Anonymous users now get per-IP rate limit keys rather than a shared "anonymous" key.
- REMAINING: The prefix is still "agent_rl" (generic). For production scale, consider namespacing as "@wekraft/agent" for clarity in Upstash analytics.
- Also note: chatbotRatelimit and chatbotGlobalRatelimit are separate limiters — correct pattern.

---

## OPEN Issues — Still Require Fixes

### Open 6: getUniqueTags O(n) Reactive Scan (HIGH — OPEN)
File: convex/workspace.ts, lines 522-542

Current code:
`	s
const tasks = await ctx.db
  .query("tasks")
  .withIndex("by_project", (q) => q.eq("projectId", args.projectId))
  .collect(); // No .take() limit — loads ALL tasks

tasks.forEach((task) => {
  if (task.type) tagsMap.set(task.type.label, task.type);
});
`

Problem: This is a Convex query (reactive subscription). It fires on EVERY task update, loading all project tasks into memory just to extract a small tag set. O(n) per reactive update.

Proposed Fix: Denormalize tags into a projectTags table.

`	s
// convex/schema.ts — add:
projectTags: defineTable({
  projectId: v.id("projects"),
  label: v.string(),
  color: v.string(),
  updatedAt: v.number(),
}).index("by_project", ["projectId"]),

// On task create/update — upsert the tag
// On task delete — check if any other task uses the tag; if not, remove it

// getUniqueTags becomes:
const tags = await ctx.db.query("projectTags")
  .withIndex("by_project", (q) => q.eq("projectId", args.projectId))
  .collect(); // O(1) — small number of unique tags
`

### Open 7: Channel List Not Cached (MEDIUM — OPEN)
File: src/app/api/teamspace/channels/route.ts

Problem: Every channel sidebar render and navigation event runs a complex 3-subquery SELECT against Turso (channel data + unread_count subquery + mention_count subquery). In a project with 50 concurrent users, this fires 50+ times per second.

Proposed Fix:
`	s
import { redis } from "@/lib/redis";

const cacheKey = channels::;
const cached = await redis.get(cacheKey);
if (cached) return NextResponse.json({ channels: JSON.parse(cached as string) });

// Run SQL...
const result = await turso.execute({ sql: querySql, args: [...] });

// Cache for 30 seconds — short enough to stay fresh, long enough to absorb spikes
await redis.set(cacheKey, JSON.stringify(result.rows), { ex: 30 });
// Invalidation: call redis.del(channels::) on channel create/update
`

### Open 8: getUserByClerkToken Full Table Scan (MEDIUM — OPEN)
File: convex/user.ts, lines 133-144

Current code:
`	s
// Fallback — loads ALL users into memory
const all = await ctx.db.query("users").collect();
const matched = all.find(
  (u) => u.clerkToken === args.clerkToken || u.clerkToken.endsWith(|),
);
`

Problem: If the exact by_token index lookup fails, every user record is loaded into memory for suffix matching. Will time out as user base grows.

Proposed Fix: Enforce that all clerkToken values are stored in their full provider|id format (e.g., oauth_github|user_xyz) on insert. Remove the suffix-matching fallback entirely. The by_token index will always hit.

### Open 9: deleteOldNotifications Unindexed Cron (MEDIUM — OPEN)
File: convex/notifications.ts, lines 724-742

Current code:
`	s
const old = await ctx.db
  .query("notifications")
  .filter((q) => q.lt(q.field("createdAt"), cutoff)) // Full table scan!
  .collect();
`

Problem: .filter() in Convex does NOT use indexes — it performs a full table scan. The notifications table will grow rapidly in production (10 event types per project action). The daily cron will time out.

Proposed Fix:
`	s
// In convex/schema.ts — notifications table already has:
// .index("by_recipient", ["recipientId", "createdAt"])
// Add a dedicated cleanup index:
.index("by_created_at", ["createdAt"])

// In convex/notifications.ts:
const old = await ctx.db
  .query("notifications")
  .withIndex("by_created_at", (q) => q.lt("createdAt", cutoff))
  .collect();
`

---

## Turso Schema — Current Table Inventory

| Table | Purpose | Indexes |
|---|---|---|
| ts_channels | Chat channels per project | idx_channels_project |
| ts_messages | All chat messages | idx_messages_channel, idx_messages_thread, idx_messages_expires |
| ts_messages_fts | Full-text search virtual table (porter tokenizer) | fts5 index |
| ts_reactions | Emoji reactions on messages | UNIQUE(message_id, user_id, emoji) |
| ts_poll_votes | Poll option votes | UNIQUE(message_id, option_id, user_id) |
| ts_notifications | Teamspace @mention notifications | idx_notifications_user |
| ts_settings | Per-project teamspace settings | PRIMARY KEY project_id |
| ts_channel_reads | Per-user per-channel last read timestamp | PRIMARY KEY (user_id, channel_id) |
| ts_private_channel_members | Private channel access control | idx_pcm_channel, idx_pcm_user |

Key schema features:
- ON DELETE CASCADE on all FOREIGN KEY references to ts_channels(id) and ts_messages(id).
- expires_at column on ts_messages enables Turso native row expiry (PRAGMA turso_enable_expiry = ON).
- made_public_at column on ts_channels tracks privacy conversion timestamp for pre-conversion message hiding.
- ts_messages_fts maintained by 3 triggers: AFTER INSERT, AFTER DELETE, AFTER UPDATE.

---

## Recommended Fix Priority (Remaining Open Issues)

1. Add by_created_at index to notifications table — 2 lines in schema.ts, eliminates cron timeout risk.
2. Remove getUserByClerkToken suffix fallback — enforce strict token format, eliminate O(n) user scan.
3. Add 30-second Redis cache to channel list route — eliminates repeated 3x SQL subquery pattern.
4. Denormalize project tags into projectTags table — eliminates O(n) reactive subscription on every task update.
