# Lattice — Product Requirements Document (PRD)

> Companion to [`SPEC.md`](SPEC.md) (the engineering *how*). This document is the product
> *why* and *what*: the problem, the users, the principles, the feature requirements, and the
> acceptance criteria. Read this first; read SPEC second.

**Status:** V1 prototype, working. **Last updated:** 2026-06-20. **Owner:** founder/PM (single-maintainer).

---

## 1. Problem

People who run small teams (founders, agency leads, PMs) carry an invisible, unmanageable **memory burden**: who promised what, who's blocked, what changed direction, which weak signals mean something is off. Existing tools *store what happened* but don't *understand what's happening*:

- **Linear / Jira** — ticket graveyards. Most of a week isn't shaped like tickets.
- **Notion** — rots the moment two people stop agreeing on structure.
- **Slack** — a firehose; state is buried in scrollback.

The founder already sees what's in front of them. The expensive gap is **what they're missing**: a commitment that quietly went stale, a blocker nobody owns, execution drifting from stated intent.

## 2. Product vision

**An AI-native tool that behaves like a quiet chief of staff who's been paying attention.** You tell it what's happening in plain language; it maintains a live, structured model of the team and tells you what you'd otherwise miss. It replaces HR-style follow-ups ("how's that going? still blocked? who owns this?"), is more intuitive than a Kanban board, understands where the team is headed, and surfaces drift and risk before they bite.

**One-line pitch:** *AI-native team execution memory — natural language in, structured team state and proactive nudges out.*

## 3. Target users

| User | Need | How Lattice serves it |
|---|---|---|
| **Founder / founder-PM** (primary) | See risk and drift without chasing people. | Morning Brief, Nudges, Ask Lattice, goal-drift detection. |
| **Team lead / PM** | Track commitments and blockers without a ticket bureaucracy. | Commitments tab, owner reassignment, interventions. |
| **Individual contributor** | Report status and push back honestly without ceremony. | Unified chat, "Can't do" (defer/plan-changed/decline), self-profile. |

Non-goal users: large orgs needing sprint planning, time-tracking, or formal project hierarchy. Lattice is deliberately small and opinionated.

## 4. Principles (the non-negotiables)

1. **The ontology is load-bearing.** Seven primitives — `intent, commitment, blocker, request, reminder, shift, signal` — are the entire vocabulary. No tickets. Getting these right matters more than the UI.
2. **Natural language is the interface.** Voice or text; the system interprets intent and routes it. The user never fills a structured form to create a commitment.
3. **One voice everywhere.** A single dry, observant chief-of-staff persona across interpret, ask, and brief. Never two prompts that drift.
4. **Derive on read before you store.** Nudges, the Morning Brief, and delivery stats are pure functions over the state graph — no cron, no second source of truth, no sync drift.
5. **Confidence is not progress.** The % on a commitment is the model's belief the work will land, never a completion bar. Label it honestly.
6. **Never guess ownership.** "I/me/self" is not an assignment. Unowned work stays unowned until someone is named — visibly, so it can be fixed.
7. **Honest pushback over false certainty.** The assistant says what's missing and challenges stale decisions; it never invents state to seem helpful.
8. **Surface the fake before the real.** Demo-seed data makes iteration fast — at the cost of fictional owners leaking, so they're clearly demo data.

## 5. Core user journeys

1. **Capture an update.** User taps the orb (or types) → "Priya's blocked on the vendor API, demo's now Friday." → Lattice records a blocker (owner Priya) and shifts the demo commitment's due date to Friday, logs change events, replies in one or two sentences. **Acceptance:** a blocker and a dated commitment exist; the Timeline shows `blocker_emerged` + `deadline_move`; no owner was invented for unstated work.
2. **Ask a question.** "Who's overloaded?" → answered from state, naming people and counts, in persona voice. **Acceptance:** the answer cites real owners/blockers from the current team; if state can't answer, it says so in one sentence rather than inventing.
3. **Morning catch-up.** Open Pulse → Morning Brief shows ≤3 bullets each of *what changed / what's at risk / what needs a decision*; Nudges list overdue/stale commitments, unowned blockers, unrevisited assumptions. **Acceptance:** every bullet traces to a real row; deferred items don't appear.
4. **Respond to a nudge.** Owner clicks Reply on "Demo video is 2 days past deadline" → chat pre-filled → "slipped, moving to Monday" → due date updates. Or "Can't do" → defer/plan-changed/decline + reason → nudges stop. **Acceptance:** deferral suppresses future nudges; the reason is stored.
5. **Run the team.** Admin invites members (email or shareable join link), reassigns owners, emails everyone their current tasks, reviews join requests. **Acceptance:** only owners/admins can do admin actions; members can edit their own profile and act on their own items.

## 6. Feature requirements

### 6.1 Capture & interpretation (P0)
- Single chat surface for both **ask** and **tell**; intent classified client-side (question vs statement) and routed to the answer vs interpret pipeline.
- Voice capture via mic → transcription → same chat path. Must handle Safari's `audio/mp4`.
- Interpretation produces structured entities + change events + optional goal shift / assumptions / interventions, and applies them immediately.
- **Owner & due-date extraction rules** enforced in the prompt (never default owner to reporter; resolve relative dates to absolute UTC; never invent a due date).
- Graceful degradation: with no OpenAI key, a deterministic keyword parser still produces reasonable primitives.

### 6.2 Live team model (P0)
- Active **goal** with confidence and a confidence **sparkline**; goal history via supersession chain.
- **Commitments** view grouped by primitive type, sorted by due urgency; per-row actions: Done, Resolve (blockers), Set due, Can't-do (defer/scope-change/decline + reason), Drop, reassign owner.
- **Timeline** of all change events; **Plan vs reality** (drift) view; pattern detection (overloaded owners, recurring blocker themes).
- **Assumptions** the team operates on, with state (holds/at-risk/invalidated/reconfirmed); invalidated assumptions raise risk.
- **Interventions** — AI-suggested next actions with urgency; accept/dismiss/mark-acted.
- **Realtime**: any teammate's change reflects in others' views within ~1s, with a brief visual flash.

### 6.3 Proactive surfaces (P0, derive-on-read)
- **Morning Brief**: changed / at-risk / needs-decision, ≤3 each, optionally LLM-polished.
- **Nudges**: overdue commitments, stale commitments (72h), open blockers (24h), stale assumptions (7d), overdue reminders — prioritized; deferred items excluded; Reply + Snooze.
- **Member delivery stats**: shipped count, open/overdue, on-time rate, avg time-to-deliver — computed from change events, fed into AI owner suggestions.

### 6.4 Teams, roles, membership (P0)
- Multi-tenant teams; create/switch teams; first-run gate (create or accept invite).
- Roles owner/admin/member; RLS-enforced. Admin: invite (email or join link), revoke, change roles, remove, email task digest, review join requests.
- Self-serve profile (name, skills, focus, bio) editable by the member only.
- Public **invite** (token) and **join** (link → request → approval) flows.

### 6.5 Persona & answering (P0)
- Shared `LATTICE_PERSONA` voice; `APP_KNOWLEDGE` lets the assistant explain its own primitives and features in-voice.
- Answers ground strictly in state; admit ignorance rather than hallucinate.

### 6.6 Platform / ops (P1)
- Platform-admin analytics dashboard (per-team health, activity trends, intervention pipeline) — gated to a single hard-coded admin email.
- In-app feedback capture (DB, with email fallback).
- Demo-seed and simulate-teammate helpers for demos (clearly non-production).

### 6.7 Explicitly out of scope (for now)
- Push delivery to Slack/email of nudges/briefs (no cron yet — the known next step).
- Tickets, sprints, story points, time tracking, Gantt/Kanban boards.
- The legacy spatial "Team Field" canvas (cut to a secondary idea; coords preserved, not rendered).
- Mobile-native apps; SSO/SAML; fine-grained per-object permissions beyond team roles.

## 7. Non-functional requirements

- **Security:** every data path enforced by Postgres RLS; service-role key used only server-side for un-membered flows (invite accept, join links/requests, admin overview, email enrichment). Bearer-token auth on every protected route.
- **Resilience:** AI is optional; all language features fall back deterministically (except audio transcription, which requires a key).
- **Performance:** single-page client; state fetched per team with a stale-response guard; realtime re-pulls full state (acceptable at prototype scale; incremental patching is a future optimization).
- **Correctness of derived views:** nudges/brief/stats must never contradict the underlying rows; deferred/closed items must be excluded consistently.
- **Privacy:** team data isolated by `team_space_id`; members never see other teams.

## 8. Success metrics (proposed)

- **Activation:** % of new users who create/join a team and log ≥1 real update in week 1.
- **Capture habit:** updates logged per active user per week (target: the tool replaces the manual "how's it going" loop).
- **Proactive value:** nudge reply/act rate; brief opens per active user.
- **Trust:** rate of "answer was wrong/invented" reports → near zero (persona's anti-hallucination stance is a feature, not a nicety).
- **Retention:** weekly active teams retained at 4 weeks.

## 9. Key risks & mitigations

| Risk | Mitigation |
|---|---|
| Spatial-canvas-style "beautiful but unusable" surfaces | List-first UI; canvas cut after persona-critique convergence. |
| Confidence read as "% done" | Labeled `conf` everywhere; honored from AI not hardcoded. |
| AI invents owners / defaults to reporter | Explicit prompt rules + inline reassignment UI. |
| No push delivery yet | Accepted P1 gap; derive-on-read keeps it trivially addable later. |
| Demo data leaking into real answers | Seed owners are clearly fictional; reset wipes team-scoped rows. |
| Persona drift across prompts | Single shared `LATTICE_PERSONA`/`APP_KNOWLEDGE` constants. |
| Repo/DB schema drift (added columns not in migrations) | Documented in SPEC §3; rebuild must commit those columns as a migration. |

## 10. Open questions / next moves

1. **Delivery layer** — Slack/email push for nudges and the Morning Brief (requires a scheduler; first real cron).
2. **Unify branding** — remove residual "OrgMind" strings in email subjects/CTAs.
3. **Commit the drifted columns** — make `supabase/migrations/` reproduce the live DB exactly.
4. **Incremental realtime** — patch state instead of full re-pull as teams grow.
5. **Revisit the spatial lens** — only if a real use emerges; coords are preserved.
