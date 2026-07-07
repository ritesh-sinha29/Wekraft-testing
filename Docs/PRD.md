# Product Requirements Document (PRD) — WeKraft SaaS
**AI-Powered GitHub-Native Collaborative Project Management Platform**
> Last Updated: June 25, 2026 — Reflects full codebase audit of v0.1.0

---

## 1. Product Vision

WeKraft is an AI-first, GitHub-native project management and team collaboration workspace designed for modern developer and product teams. It replaces disconnected, passive tracking tools (like Jira + Slack + Linear) with a unified, agentic platform where two AI teammates — **KAYA** (AI Product Manager) and **HARRY** (AI Senior Developer) — operate alongside the team inside the same interface.

**Core Mission:** Eliminate developer administrative overhead by embedding autonomous AI directly into the project management workflow, so teams spend time building — not updating boards.

---

## 2. Target Users

| Persona | Description |
| :--- | :--- |
| **Student Developer Teams** | Engineering students in incubators (GUSEC, i-Hub Gujarat) needing structured project management without prior PM expertise. |
| **Early-Stage Startups** | 2–8 person teams shipping their first product. Need speed, AI assistance, and integrated comms without enterprise overhead. |
| **Software Agencies** | Small agencies managing multiple client projects who benefit from per-project Kanban, sprints, and client-facing service desks. |
| **Solo Developers** | Individual builders who want KAYA to handle sprint planning and scheduling so they can stay in flow. |

---

## 3. Core Feature Areas

### 3.1. Project & Task Management

**Backlog (List View)**
- Create and manage tasks with title, description, custom tags, priority (high/medium/low), status (not started -> inprogress -> reviewing -> testing -> completed), and date estimations.
- Attach files (via S3 / AWS) directly to tasks.
- Assign multiple members to tasks using a separate join table (taskAssignees) for scalable rendering.
- Filter by status, priority, assignee, sprint, and custom tag.
- Tutorial overlays (via Driver.js) for first-time users.

**Kanban Board**
- Drag-and-drop task cards across status columns powered by @dnd-kit.
- Visual priority indicators and assignee avatars on cards.

**Table View**
- Spreadsheet-like table rendering of all tasks with sortable columns.

**Issue Tracker**
- Independent issue entity separate from tasks, with fields: severity (critical/medium/low), environment (local/dev/staging/production), linked file, linked task, GitHub issue URL.
- Three issue types: manual (user-created), task-issue (promoted from a blocked task), github (imported from GitHub webhook).
- Issue Kanban board with drag-and-drop.
- Blocking mechanism: a task marked as a bug issue cannot be marked completed until the linked issue is closed.

**Sprint Management**
- Create sprints with goals, start/end dates, and status lifecycle (planned -> active -> completed).
- Add tasks and issues from backlog to active sprint.
- Sprint dashboard with progress bar, open bug count, burn rate metric, and sprint comparison bar chart.
- Final sprint stats snapshotted on completion.

---

### 3.2. Agentic AI System (KAYA and HARRY)

**KAYA — AI Product Manager**
- Context-aware AI assistant embedded in the project sidebar.
- LangGraph agentic pipeline with persistent semantic memory via Mem0 (cross-session user context).
- Tool-calling capabilities: Create tasks, issues, sprints from natural language; Set up delivery schedulers; Insert calendar events and milestones; Query project velocity and generate reports; Sprint planning with backlog analysis.
- Human-in-the-Loop Interrupt Cards: When KAYA proposes multi-step actions (e.g., creating a sprint backlog), she renders interactive approval cards in chat — the user reviews and confirms before Convex mutations execute.
- Quick templates for common PM tasks.
- Floating KAYA launcher accessible from any project page.
- Rate-limited via Upstash Redis sliding window on the /api/agent route.
- SSE streaming response for real-time AI output.

**HARRY — AI Senior Developer**
- Specialized agent focused on codebase analysis and PR review.
- Analyzes GitHub repository structure, pull request diffs, and commit histories.
- Provides code review feedback, bug identification, and architectural suggestions.
- Lives within the HeatmapFlow codebase visualization interface.

**KAYA in Teamspace**
- KAYA participates in chat channels via the /api/kaya-teamspace endpoint.
- Responds to @mentions, summarizes threads, and performs actions from within channel conversations.
- AI Teamspace toggle is per-project (canUseAITeamspace in projectDetails).

---

### 3.3. Teamspace (Real-Time Chat)

- Channels: Public, private, and announcement channels per project.
- Messaging: Rich message composer with markdown support, file/image attachments (stored in S3), emoji reactions, and threaded replies.
- Link Unfurling: Auto-previews URLs pasted into messages.
- Polls: Create and vote on polls directly in channels.
- Document Mode: Collaborative document creation within a channel.
- Search: Full-text search overlay across all channel messages.
- Member Presence: Online/offline presence indicators.
- Notification Center: Dedicated panel for real-time Convex notifications.
- Architecture: Turso edge SQL for high-throughput message storage; Ably pub/sub for real-time delivery; IndexedDB soft-expiry caching on client for offline resilience.

---

### 3.4. Team Meets (Video Calling)

- Start/join live video meetings directly within the workspace.
- Powered by Stream Video SDK.
- Meeting lifecycle: host creates meeting -> notification fanned out to all project members -> participants join -> host ends meeting.
- Meeting records stored in team_meets Convex table with participant list.
- Stale meeting self-heal: daily cron at 03:00 UTC marks meetings older than 4 hours as inactive.

---

### 3.5. Calendar and Scheduling

- Full calendar view with event and milestone types (powered by FullCalendar).
- Create, edit, delete events and milestones per project.
- KAYA can insert events autonomously after user approval via interrupt card.
- Delivery Schedulers: Configure automated delivery alert jobs per project (configurable frequency in days, recipient email). KAYA can activate and manage schedulers via tool calls. Runs via Inngest event engine.

---

### 3.6. GitHub Integration

- Repository Sync: Connect a GitHub repository to a project (via Octokit/GitHub App).
- Issue Import: Import GitHub issues directly into the WeKraft issue tracker.
- Webhook Integration: GitHub events (pushes, PR creation, issue events) trigger real-time updates in the platform.
- Codebase Heatmap: Visualize codebase contribution/activity patterns (ReactFlow-based @xyflow/react).
- Repository Structure Browser: Browse connected repository file tree.

---

### 3.7. Analytics and Insights Dashboard

- Sprint Bar Chart: Visual comparison of sprint completion rates across the project.
- Weekly Velocity Chart: Track tasks completed per week.
- Weekly Engagement Chart: Team activity heatmap per week.
- Member Workload Card: Visualize task distribution across team members.
- Team Contribution Radar: Multi-dimensional radar chart of team participation metrics.
- Activity Overview Card: Real-time summary of recent project activity.
- Environmental Severity Heatmap: Issue severity breakdown by environment.
- My Work Sheet: Personal task/issue view filtered to the current user.
- Activity Calendar Heatmap: GitHub-style contribution graph per user.

---

### 3.8. Time Intelligence (Project Timeline)

- Project Timeline: Visual Gantt-like chart of tasks and sprints over time.
- Delay Debt Analyzer: Quantifies delays and their cascading impact on the project schedule.
- Milestone Trajectory: Projects future milestone completion based on current burn rate.
- Pace Tracker: Real-time pace vs. planned rate comparison.

---

### 3.9. Customer Desk (CRM Lite)

- Per-project customer registry (serviceCustomers).
- Log customer feature requests and bug reports (serviceRequests with feature_request/bug_report types).
- Request lifecycle: pending -> approved (auto-creates task/issue) / rejected.
- Role-gated: only Owner/Admin can create customers; members can log requests.

---

### 3.10. Profile and Social

- User profile with bio, up to 3 social links, and GitHub username linkage.
- Skill tags with automatic refresh tracking.
- GitHub-style activity calendar on profile.

---

### 3.11. Project Discovery and Upvotes

- Public projects are discoverable in the explore feed.
- Upvote system with denormalized counter and join table for unique-per-user enforcement.
- Project work status labels: ideation -> validation -> development -> beta -> production -> scaling.
- Invite-link-based project joining.
- Join request workflow: user sends request -> owner/admin approves/rejects -> notification fired.

---

### 3.12. Onboarding

- Multi-step onboarding flow capturing: occupation, primary usage goals, heard-from source.
- Welcome modal and getting-started checklist.
- One-time feature tutorial overlays (Driver.js) per feature: tasks, issues, sprints, time logs.
- Referral system: referalCreated (user's invite code) and referalUsing (code they used) tracked in userDetails.

---

### 3.13. IDE Extension (VS Code)

- VS Code extension connects to the WeKraft workspace.
- OAuth handshake flow: Extension opens /extension?callback_url=vscode://wekraft.wekraft/auth -> user grants access -> createHandshakeToken() generates a 5-minute one-time token -> browser redirects to vscode://wekraft.wekraft/auth?token=<hex> -> extension calls exchangeHandshakeToken({ token }) via Convex -> returns { userId, apiKey } — token deleted immediately.
- API key stored in userApiKeys table with per-key sliding-window rate limiting.
- Two-way task updates and AI context from the IDE without switching apps.

---

### 3.14. Billing and Subscription

Plan     | India (Razorpay) | Global (Stripe)
Free     | Rs. 0            | 
Plus     | Rs. 649/mo       | /mo
Pro      | Rs. 1499/mo      | /mo

- Dynamic routing: IP geolocation (ipapi.co) determines India (IN) -> Razorpay; else -> Stripe.
- Graceful cancellations: users keep access until currentPeriodEnd.
- Daily safety-net cron at 00:30 UTC auto-downgrades expired plans.
- upgradeAccount Convex mutation is internalMutation — cannot be called from browser.
- LemonSqueezy integration available as an additional/alternate payment provider.

---

### 3.15. Admin Panel

- Internal admin dashboard (/admin) gated by isAdmin: true on the user record.
- Overview of all users, plans, and platform-level metrics.
- Support query management (supportQueries table).
- Platform-wide cloud storage and Kaya usage monitoring per user.

---

## 4. Notifications System

Trigger                      | Notification Type
User joins project           | member_joined
User leaves project          | member_left
User removed                 | member_removed
New join request             | join_request
Join request accepted        | request_accepted
Join request rejected        | request_rejected
Role changed                 | role_changed
@mention in comment          | mentioned
Project delivery alert       | project_alert
Team video call started      | meeting_started

- Stored in Convex notifications table with by_recipient + by_recipient_unread compound indexes.
- Delivered via Convex reactive subscriptions to the client in real-time.
- Cleanup: daily cron at 02:00 UTC deletes notifications older than 30 days.

---

## 5. Access Control Model

Role     | Create Tasks/Issues | Use KAYA | Manage Members | Delete Project
Owner    | YES                 | YES      | YES            | YES
Admin    | YES                 | YES      | YES            | NO
Member   | Configurable        | Config.  | NO             | NO
Viewer   | NO                  | NO       | NO             | NO

*Controlled by memberCanCreate and memberUseKaya flags in projectDetails.

---

## 6. Non-Functional Requirements

Requirement         | Target
Chat Latency        | < 50ms via Turso edge SQL + Ably pub/sub
Agent Response      | Streaming SSE; first token < 1.5s
API Rate Limiting   | Sliding window via Upstash Redis (per-IP + per-API-key)
Uptime              | 99.9% (Vercel Pro + Convex Pro)
Auth Security       | Clerk JWTs, webhook HMAC (timing-safe), internalMutation gating
Error Monitoring    | Sentry (client + server + edge configs)
SEO                 | JSON-LD structured data, sitemap, robots.txt, Open Graph meta
Accessibility       | Radix UI primitives (ARIA-compliant)
Mobile              | Responsive layouts; tablet-tested

---

## 7. Open Roadmap Items

- [ ] Real burndown chart (per-day task-completion time series from backend)
- [ ] HARRY full PR review automation triggered by GitHub webhooks
- [ ] Livekit integration evaluation (currently installed alongside Stream Video)
- [ ] Enterprise plan tier
- [ ] @mention autocomplete in teamspace message composer
- [ ] Public project marketplace / discovery feed
- [ ] Mobile app (React Native or Flutter)
- [ ] Billing: Lemon Squeezy as primary global provider migration
