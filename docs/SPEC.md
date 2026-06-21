# Lattice — Engineering Specification (rebuild-from-scratch reference)

> Goal of this document: a single source of truth precise enough that an engineer
> who has never seen this repo can reconstruct Lattice — schema, server, client, AI,
> and the way every feature interacts — without reading the original code.
>
> Companion docs: [`PRD.md`](PRD.md) (the *why* and *what*), [`DOCUMENTATION.md`](DOCUMENTATION.md) (orientation), [`STORY.md`](STORY.md) (design history).

---

## 0. One-paragraph mental model

Lattice is an **AI-native team execution memory**. A user speaks or types a natural-language update ("Priya's blocked on the vendor API, demo's now Friday"). The server interprets it with an LLM into **structured state** — commitments, blockers, requests, goal shifts, assumptions, suggested interventions — and persists it to Postgres. The same chat surface answers questions ("who's overloaded?") from that state. Nudges, the Morning Brief, and per-member delivery stats are **derived on read** (pure functions over state, never stored). Everything is multi-tenant (teams) with Supabase Auth + Row-Level Security. There is **no cron, no background worker, no task queue** — state in, views computed out.

The entire operational vocabulary is **seven primitives**: `intent`, `promise` (shown as "Commitment"), `blocker`, `request`, `reminder`, `shift`, `signal`. No tickets, no sprints.

---

## 1. Stack & runtime

| Layer | Choice | Version / notes |
|---|---|---|
| Framework | Next.js (App Router) | `next@16.2.3`, Turbopack dev. Pages + Route Handlers in one `app/` tree. |
| UI | React | `react@19.2.5`, `react-dom@19.2.5`. One `"use client"` page; no design-system dep. |
| Language | TypeScript (strict) | `typescript@5.9.3`. Canonical check: `npx tsc --noEmit`. |
| Styling | Tailwind v4 PostCSS + hand-written `globals.css` | `tailwindcss@4.1.18`, `@tailwindcss/postcss`. Most styling is custom CSS variables + classes. |
| DB / Auth / Realtime | Supabase (Postgres 15) | `@supabase/supabase-js@2.103.0`. RLS on every table. |
| AI — text | OpenAI Chat Completions | `openai@6.34.0`. Default model `gpt-5.4-mini` (`OPENAI_MODEL`). JSON response-format where shape matters. |
| AI — audio | OpenAI transcription | Default `gpt-4o-mini-transcribe` (`OPENAI_TRANSCRIBE_MODEL`); `whisper-1` works too. |
| Email | nodemailer (SMTP) | `nodemailer@8.0.5`. Optional; used for invites/digests/feedback fallback. |
| Validation | zod | `zod@4.2.0` (present in deps; light usage). |
| Deploy target | Vercel | `APP_BASE_URL` / `VERCEL_PROJECT_PRODUCTION_URL` drive email links. |

**Module alias:** `@/...` → repo root (so `@/lib/v2` = `lib/v2.ts`).

### Request lifecycle

```
Browser (app/page.tsx, single client component)
  → authedFetch() attaches `Authorization: Bearer <supabase access_token>`
  → /api/... Route Handler
      → requireUserSupabaseClient(request): validates JWT, returns a Supabase
        client bound to that user (so Postgres RLS runs as them)
      → handler reads/writes via lib/* and/or calls OpenAI
      → returns JSON, typically { state, team, ... } so the client can refresh
  ← client setState; ALSO a Supabase Realtime subscription re-fetches on any DB change
```

---

## 2. Environment variables

```bash
# AI (optional — without it, deterministic fallback parsers run)
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-5.4-mini                  # interpret, ask, brief, analyze
OPENAI_TRANSCRIBE_MODEL=gpt-4o-mini-transcribe   # /api/transcribe

# Supabase (required for anything real)
NEXT_PUBLIC_SUPABASE_URL=https://<ref>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=...          # client + user-scoped server calls (RLS enforced)
SUPABASE_SERVICE_ROLE_KEY=...              # bypasses RLS; required for invite accept,
                                           # join-links, join-requests, admin overview,
                                           # member-email enrichment

# Legacy default team for unauth'd/legacy paths (superseded by auth-derived teams)
LATTICE_TEAM_SPACE_ID=demo-team-space

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_SECURE=true                           # "false" → STARTTLS
SMTP_USER=...
SMTP_PASS=...
EMAIL_FROM=...                             # falls back to SMTP_USER
APP_BASE_URL=http://localhost:3000         # prod: https://lattice-opal.vercel.app
```

Behavior degrades gracefully: **no OpenAI key** → deterministic keyword parsers for interpret/ask/brief (only `/api/transcribe` hard-fails with 503). **No service-role key** → invite/join/admin flows 500/503 and member emails aren't enriched. Env changes require a **server restart** (Next.js does not hot-reload `.env.local`).

---

## 3. Data model (authoritative = live DB + app code)

> ⚠️ **Repo/DB drift to know before you rebuild.** The repo `supabase/migrations/` does NOT
> contain the columns `field_objects.due_at`, `field_objects.deferred_until`,
> `field_objects.decline_reason`, nor `team_members.skills/focus/bio`. But the **live
> database and the application code both use them** (commitment due dates, the "can't do"
> deferral flow, and member profiles all depend on them). They were applied directly via the
> Supabase MCP without a committed migration file. Treat the columns below as REQUIRED. When
> rebuilding, add them as a migration so the repo is self-consistent.

All tables in schema `public`; **RLS enabled on every table.**

### 3.1 Enums

| Enum | Values |
|---|---|
| `orgmind_object_type` | `intent, promise, blocker, shift, request, reminder, signal` |
| `orgmind_request_state` | `draft, sent, acknowledged, resolved, denied` |
| `lattice_goal_state` | `active, paused, achieved, dropped, superseded` |
| `lattice_change_kind` | `goal_shift, scope_change, priority_change, deadline_move, owner_change, blocker_emerged, blocker_resolved, assumption_invalidated, confidence_change, commitment_added, commitment_completed, commitment_stale` |
| `lattice_assumption_state` | `holds, at_risk, invalidated, reconfirmed` |
| `lattice_intervention_state` | `suggested, accepted, dismissed, acted` |
| `lattice_team_role` | `owner, admin, member` |
| `lattice_invite_state` | `pending, accepted, revoked, expired` |
| `lattice_join_request_state` | `pending, approved, rejected, cancelled` |

Create each idempotently: `do $$ begin create type ...; exception when duplicate_object then null; end $$;`

### 3.2 Tables

Every `id` is `text` unless noted. Every FK to `team_spaces(id)` is `ON DELETE CASCADE`. Every FK to `auth.users(id)` is `ON DELETE SET NULL` unless noted CASCADE.

**`team_spaces`** — one row per team.
- `id text PK` (default `gen_random_uuid()::text`; app overrides with a slug), `name text not null`, `active_intent text not null default 'Ship a demo people trust'`, `tensions text[] not null default '{}'`, `broadcast text[] not null default '{}'`, `created_at`, `updated_at`.
- `created_by uuid → auth.users(id)` (nullable).
- `join_token text not null` default `replace(gen_random_uuid()::text,'-','')`; **unique index** `team_spaces_join_token_idx`. (Public join links resolve teams by this token, falling back to `id`.)

**`team_members`** — membership per (user, team).
- `id text PK` (app format `m-<teamSpaceId>-<userId.slice(0,8)>`), `team_space_id → team_spaces CASCADE`, `user_id uuid → auth.users CASCADE` (nullable for V1 seed rows), `name text not null`, `role lattice_team_role not null default 'member'`, `created_at`.
- `skills text[] not null default '{}'`, `focus text`, `bio text`.
- Partial unique index `(team_space_id, user_id) where user_id is not null`.
- ⚠️ Role-column caveat: the original V1 `schema.sql` declared `role text`; migration 2's `add column if not exists role lattice_team_role` is a no-op if a `text role` already exists. Intended end state is the **enum** column. Build it as the enum from the start.

**`field_objects`** — the single table holding all seven primitives.
- `id text PK`, `team_space_id → team_spaces CASCADE`, `type orgmind_object_type not null`, `title text not null`, `detail text not null`, `owner text` (matches `team_members.name`, case-insensitively, when set), `status text` (free-form: `new/in_progress/done/resolved/dropped/...`), `confidence numeric not null default 0.7 CHECK 0..1`, `position_x numeric default 50 CHECK 0..100`, `position_y numeric default 50 CHECK 0..100` (legacy spatial coords; preserved, not rendered), `pulse text not null default 'quiet' CHECK in (quiet,active,tense,stale,clear)`, `links text[] default '{}'`, `created_at`, `updated_at`.
- `due_at timestamptz`, `deferred_until timestamptz`, `decline_reason text`.

**`memory_events`** — chronological memory feed. `id`, `team_space_id`, `kind orgmind_object_type` (nullable; broadcast/follow-up stored as null), `text not null`, `created_at`.

**`delegated_requests`** — the `request` primitive broken out. `id`, `team_space_id`, `target not null`, `ask not null`, `why not null`, `state orgmind_request_state default 'draft'`, `linked_to`, timestamps.

**`reminders`** — self-nudges. `id`, `team_space_id`, `text not null`, `trigger text not null`, `linked_to`, `resolved_at`, timestamps.

**`interpretations`** — raw log of every NL update. `id`, `team_space_id` (nullable), `raw_input not null`, `reply not null`, `entities jsonb default '[]'`, `follow_up_question`, `broadcast jsonb default '[]'`, `created_at`.

**`goals`** — first-class goals. `id text PK` (app: `goal-<ts>`, no default), `team_space_id`, `title not null`, `detail`, `state lattice_goal_state default 'active'`, `priority int default 1`, `confidence numeric default 0.7 CHECK 0..1`, `previous_goal_id text → goals(id) ON DELETE SET NULL` (history chain), timestamps.

**`change_events`** — append-only log of every meaningful transition. `id` (`chg-<ts>`), `team_space_id`, `kind lattice_change_kind not null`, `summary not null`, `detail`, `target_id`, `target_type`, `previous_value jsonb`, `new_value jsonb`, `source` (`interpretation`/`manual`/`seed`), `reported_by uuid → auth.users SET NULL`, `impact jsonb` (`{ teamReadable?, affects?[] }`), `created_at`. No `updated_at`.

**`assumptions`** — beliefs the team operates on. `id` (`assum-<ts>-<i>`), `team_space_id`, `statement not null`, `state lattice_assumption_state default 'holds'`, `tied_to`, `last_checked_at`, timestamps.

**`dependencies`** — explicit dependency links (table exists, unused in UI). `id`, `team_space_id`, `source_id not null`, `target_kind not null`, `target_ref not null`, `note`, `resolved_at`, `created_at`.

**`confidence_signals`** — time series of confidence readings (powers the sparkline). `id` (`sig-<ts>`), `team_space_id`, `target_id not null`, `target_type not null`, `confidence numeric not null CHECK 0..1` (no default), `note`, `reported_by uuid → auth.users SET NULL`, `created_at`.

**`interventions`** — AI-suggested next actions. `id` (`int-<ts>-<i>` / `int-auto-<ts>-<i>`), `team_space_id`, `title not null`, `rationale not null`, `action_kind text not null`, `urgency int default 2 CHECK 1..5`, `target_id`, `target_type`, `state lattice_intervention_state default 'suggested'`, `dismissed_at`, `acted_at`, timestamps.

**`team_invitations`** — email invites. `id` (`inv-<ts>-<hex>`), `team_space_id`, `email not null`, `token text not null UNIQUE` (`base64url`, 24 random bytes), `role lattice_team_role default 'member'`, `state lattice_invite_state default 'pending'`, `invited_by`/`accepted_by uuid → auth.users SET NULL`, `expires_at timestamptz not null default now()+'7 days'`, timestamps. Indexes on `(team_space_id)` and `(lower(email))`.

**`team_join_requests`** — request-to-join via public link. `id` (`join-<ts>-<hex>`), `team_space_id`, `user_id uuid → auth.users CASCADE`, `name`, `email`, `message`, `state lattice_join_request_state default 'pending'`, `reviewed_by uuid → auth.users SET NULL`, `reviewed_at`, timestamps. **UNIQUE `(team_space_id, user_id)`**. Indexes `(team_space_id, state, created_at desc)`, `(user_id, state)`.

**`platform_feedback`** — product feedback. `id uuid PK default gen_random_uuid()`, `user_id uuid → auth.users CASCADE not null`, `email`, `message not null`, `created_at`. Index `(created_at desc)`.

### 3.3 Functions (all `security definer`, `set search_path = public`, except `set_updated_at`)

- `set_updated_at()` → trigger: `new.updated_at = now(); return new;` (invoker rights). Attached BEFORE UPDATE on `team_spaces, field_objects, delegated_requests, reminders, goals, assumptions, interventions, team_invitations, team_join_requests`.
- `is_orgmind_team_member(target text) → bool`: `exists(select 1 from team_members where team_space_id=target and user_id=auth.uid())`. The membership predicate used by almost every RLS policy.
- `is_lattice_team_admin(target text) → bool`: same but `and role in ('owner','admin')`.
- `is_platform_admin() → bool`: `lower(email)='maantech123@gmail.com'` for `auth.uid()` (hard-coded super-admin).
- `handle_new_orgmind_user()` → trigger: inserts the new auth user into `demo-team-space` as a member. **Its trigger `on_auth_user_created_add_orgmind_member` is created in V1 then DROPPED by migration 2** — so in the final system new signups do NOT auto-join the demo team; they go through `FirstTeamGate` (create or accept invite). Keep the function, drop the trigger.

> Note: there is **no** `team_members` invariant trigger raising SQL error `42501`. Admin-only mutation of members is enforced purely by RLS (`team_members_admin_update/delete/insert`). (Some prose elsewhere references such a trigger; it does not exist in this codebase.)

### 3.4 RLS policies (summary; `M(x)` = `is_orgmind_team_member(x)`, `A(x)` = `is_lattice_team_admin(x)`)

- **Member-scoped tables** (`field_objects, memory_events, delegated_requests, reminders, goals, change_events, assumptions, dependencies, confidence_signals, interventions`): SELECT/INSERT (and UPDATE where mutable) gated by `M(team_space_id)`. `change_events` and `confidence_signals` are append-only (read + insert only).
- **`team_spaces`**: read/update require `M(id)`; INSERT requires `auth.uid() is not null`.
- **`team_members`**: read requires `M(team_space_id)`; UPDATE/DELETE require `A(team_space_id)`; INSERT allowed if `user_id=auth.uid()` (self-join) OR `A(team_space_id)` (admin adds).
- **`interpretations`**: `team_space_id is null OR M(team_space_id)` for read & insert.
- **`team_invitations`**: read `M`; insert/update `A`.
- **`team_join_requests`**: read `auth.uid()=user_id OR A`; insert `auth.uid()=user_id AND state='pending'`; update `A`.
- **`platform_feedback`**: self-insert (`user_id=auth.uid()`), self-read, plus `is_platform_admin()` read & delete.

**Authorization model takeaway:** app code mostly does NOT re-check permissions — it runs queries *as the user* and lets RLS reject them. `lib/teams.ts` infers "denied" from a 0-row result on RLS-scoped UPDATE/DELETE. Explicit app-level checks exist only for the **commitment owner-gate**, **digest admin-gate**, and **join-request review admin-gate** (see §5).

### 3.5 Realtime publication

Added to `supabase_realtime`: `change_events, interventions, goals, field_objects, assumptions, team_members, team_invitations`. The client subscribes to a subset (see §6.4).

### 3.6 Seed data

`schema.sql` seeds `demo-team-space` ("Hackathon Demo Team") with 3 members (Aryan/Meera/Team lead), 5 field objects, 3 memory events, 1 request, 1 reminder, plus tensions/broadcast. Migration 2 promotes the first demo member to `owner`. This is optional for a fresh build; the live product creates teams per-user.

---

## 4. The ontology (load-bearing — the seven primitives)

Every `field_objects.type` is exactly one of these. The same table appears in the UI tooltips (`typeDescription`/`typeExample` in `page.tsx`), the AI ontology (`APP_KNOWLEDGE` in `lib/persona.ts`), and the interpret JSON schema (`lib/ai-v2.ts`). **Change one, change all four.**

| Type | Meaning | Extra fields | Example |
|---|---|---|---|
| `intent` | What the team is trying to do — a direction, not a task. | title, detail | "Ship a demo people trust by Friday." |
| `promise` (UI: **Commitment**) | A concrete deliverable someone agreed to. | owner, status, `due_at`, confidence | "Demo video — know2, due Fri, 80% confidence." |
| `blocker` | Something stopping progress; open until resolved/dropped. | owner, status | "Vendor API is down — Priya." |
| `request` | An ask from one person to another, not yet accepted. | target, state (draft/sent/acknowledged/resolved/denied) | "Ask legal to review the data policy." |
| `reminder` | A self-nudge tied to a time/trigger. | trigger | "Remind me at 8pm to retry the deploy." |
| `shift` | A direction/scope pivot. | — | "Dropping analytics — focus is the demo." |
| `signal` | A weak observation worth remembering, not yet actionable. | — | "Legal has been quiet for two weeks." |

**Confidence ≠ progress.** The number on a commitment is *Lattice's confidence the work will land*, not a % done. Default `0.7` when unstated. UI labels it `conf`. This distinction is enforced in the persona prompt and the UI label.

---

## 5. Server — API surface (every route)

All `/api/v2/*` routes require auth via `requireUserSupabaseClient` (→ **401** missing/expired token, **503** Supabase unconfigured) unless marked public. "Resolve team" = `getUserActiveTeam(supabase, userId, body.teamId)`; no team → **400 `No team.`** (exact strings vary, noted inline). Standard success carries `{ state, team }` so the client can refresh; standard failure is **500** with an `{ error }` string.

### 5.1 Chat, state, AI

**`POST /api/v2/interpret`** — NL update → structured state. Body `{ input: string (req), apply?: boolean, teamId? }`. Empty input → 400. Fetches state + `listTeamMembers` (members default `[]`), runs `interpretV2(input, state, members)`. If `!apply` → returns `{ interpretation, state: current, team }` (no writes). If `apply` → `applyInterpretationV2ToDatabase` persists everything (new field objects, memory, requests, reminders, an `interpretations` row, team_spaces update; detected change_events; goal shift; assumptions; interventions; confidence impact) and returns the next state. Errors → 500 `Interpretation failed.`

**`POST /api/v2/ask`** — answer a question from state. Body `{ query (req), teamId?, history?: {role,content}[] }`. History filtered to valid user/assistant turns, last **6**, each content ≤**800** chars. Builds a text state context (goal, team confidence, at-risk count, commitments, blockers, assumptions, recent 8 changes, structural overload/drift). System messages: `LATTICE_PERSONA` + answering rules, then `APP_KNOWLEDGE`, then the state block; then history; then the query. OpenAI temp **0.6**. No key → deterministic in-voice fallback. Returns `{ answer, state, team }`. Errors carry a `stage` (`auth/body/team/state/openai`) in the message; 500.

**`POST /api/v2/brief`** — deterministic Morning Brief. Body `{ teamId?, sinceHours? }` (clamped `[1,336]`, default **72**). `buildBrief` produces three ≤3-item lists: `changed` (recent change_events deduped by summary), `atRisk` (open blockers + low-confidence <0.5 open promises + at-risk/invalidated assumptions + an overloaded owner + drift count), `needsDecision` (top interventions by urgency + low-confidence <0.4 active goal + an `atRiskCount ≥ 4` triage line). If OpenAI present and any bullets exist, `refineWithAI` polishes them (temp **0.3**, JSON); errors keep raw. Returns `{ brief, generatedAt, sinceHours, team }`.

**`GET /api/v2/nudges?team=`** — derived check-ins. `deriveNudges(state).slice(0,6)`. Read-only. Returns `{ nudges, team }`. (Formulas in §7.)

**`GET /api/v2/state?team=`** — current `LatticeState`. If the user has no team, returns **200** `{ state:null, team:null, teams:[] }` (not an error). Else `{ state, team }`.

**`POST /api/v2/analyze`** — recompute interventions. Body `{ teamId? }`. Builds candidate interventions from: AI goal-alignment (`aiGoalAlignment`, temp 0.2, JSON; falls back to `goalDrift`) when ≥2 drifting; overloaded owners (≥2 open blockers); recurring blocker tokens (first 2); stale commitments (promise, confidence <0.6, pulse-as-date older than 7d); at-risk assumptions. Dedups against currently-open intervention titles, inserts the rest (`state: suggested`, id `int-auto-<ts>-<i>`). Returns `{ state, team, added }`.

**`POST /api/transcribe`** — multipart `audio` (a `File`) → `{ text }`. Requires `OPENAI_API_KEY` (else **503**). 400 on missing/empty file. Calls `openai.audio.transcriptions.create({ file, model: OPENAI_TRANSCRIBE_MODEL ?? "gpt-4o-mini-transcribe" })`. Error responses include `{ upstreamStatus, file, model }`; status 400 if upstream 4xx else 500. No DB writes.

### 5.2 Entity mutations

**`PATCH /api/v2/commitment`** — the workhorse. Body `{ id (req), action (req), confidence?, owner?, dueAt?, deferredUntil?, reason?, teamId? }`. Loads the `field_objects` row (404 if absent).
- **Owner gate:** every action *except* `set_confidence` is owner-gated. If caller's team role is not `owner`/`admin`, the route looks up the caller's `team_members.name` and requires it to equal the item's `owner` (both trimmed/lowercased, non-empty) — else **403 `Only the current owner (or a team admin) can change this.`**
- **Actions** (each writes a `field_objects` UPDATE + a `change_events` INSERT, `source:"manual"`, `id chg-<ts>`):

| action | patch | change kind | summary |
|---|---|---|---|
| `complete` | status=done, pulse=clear, confidence=1 | `commitment_completed` | `Completed: <title>` |
| `resolve` | status=resolved, pulse=clear | `blocker_resolved` (if blocker) else `commitment_completed` | `Resolved: <title>` |
| `drop` | status=dropped, pulse=quiet | `scope_change` | `Dropped: <title>` |
| `set_confidence` | confidence=`confidence` (number req, else 400) | `confidence_change` | `Confidence on "<title>" now N%` |
| `set_owner` | owner=trimmed string or null | `owner_change` | reassigned/unassigned |
| `set_due` | due_at=ISO(dueAt) or null | `deadline_move` | due date / cleared |
| `defer` | deferred_until=ISO (req, else 400), decline_reason=reason, pulse=stale | `commitment_stale` | `"<title>" deferred until <date>` |
| `decline` | status=dropped, pulse=quiet, decline_reason=reason | `scope_change` | `"<title>" declined` |
| `scope_change` | pulse=tense, decline_reason=reason | `scope_change` | `"<title>" flagged: plan changed` |
| (other) | — | — | **400 `Unknown action.`** |

**`POST /api/v2/goal`** — set active goal. Body `{ title (req), detail?, confidence?, teamId? }`. Supersedes current active goal, inserts new active goal (`goal-<ts>`, priority 1, confidence `?? 0.7`, `previous_goal_id` = prior), logs `goal_shift`. **`PATCH /api/v2/goal`** — edit. Body `{ id (req), state?, confidence?, title?, detail?, teamId? }`; patches provided fields; no change_event.

**`PATCH /api/v2/intervention`** — Body `{ id (req), state (req ∈ suggested/accepted/dismissed/acted), teamId? }`. Sets `state` (+ `dismissed_at`/`acted_at` timestamps for those states). Returns `{ state, team }`.

**`PATCH /api/v2/assumption`** — Body `{ id (req), state (req ∈ holds/at_risk/invalidated/reconfirmed), teamId? }`. Sets `state` + `last_checked_at=now`. No change_event.

### 5.3 Teams, membership, invites, join

- **`GET /api/v2/teams`** → `{ teams }` (caller's memberships). **`POST`** `{ name (req) }` → creates `team_spaces` (id = `slugify(name).slice(0,40)-<hex6>`, active_intent `"Set your first goal"`) + owner `team_members` row.
- **`GET /api/v2/teams/[teamId]/members`** → members with emails (service-role enrichment via `auth.admin.getUserById`). **`PATCH`** `{ memberId, role }` (RLS admin). **`DELETE ?memberId=`** (RLS admin).
- **`PATCH /api/v2/teams/[teamId]/members/profile`** — self-edit only. Body `{ name?≤60, skills?≤20 each≤40, focus?≤200|null, bio?≤600|null }`. Looks up caller's own row; sanitizes; updates only their row (RLS + the lib's 0-row → throw guard). 403 if not a member.
- **`GET/POST/DELETE /api/v2/teams/[teamId]/invites`** — list pending; create `{ email (must contain "@"), role? }` (admin via RLS, mapped to 403 on failure); revoke `?inviteId=`.
- **`POST /api/v2/invitations/accept`** — **service-role** (invitee isn't a member yet). Body `{ token }`. Validates invite (pending + not expired), upserts membership at the invite's role, marks invite accepted. 400 on any failure.
- **`GET /api/v2/join-links/[token]`** — **PUBLIC** (no auth). Service-role resolves `team_spaces` by `join_token` (fallback `id`). Returns `{ team: { teamSpaceId, teamName } }` or 404.
- **`POST /api/v2/join-requests`** — service-role. Body `{ token, message? }`. Already a member → `{ alreadyMember:true }`. Else creates a `team_join_requests` row (pending). Falls back to encoding as a special `team_invitations` row (`joinreq:<uid>:<token>`) if the join-requests table is missing.
- **`GET/PATCH /api/v2/teams/[teamId]/join-requests`** — list pending; review `{ joinRequestId, action: approve|reject }`. **Explicit admin check**: reads caller's role; non-owner/admin → 403. Approve inserts a `member` row + marks `approved`; reject marks `rejected`.
- **`POST /api/v2/teams/[teamId]/digest`** — email each member their active items. Requires `isEmailConfigured()` (else 503). **Explicit admin gate** (team role owner/admin, else 403). Sends one email per member-with-email; returns `{ ok, sent, skipped }`.

### 5.4 Platform / demo

- **`GET /api/v2/admin/overview`** — **platform-admin only** (`isPlatformAdminEmail`, else 403). Service-role reads across ALL teams; computes per-team health snapshots (healthScore 8–98, status healthy/watch/at-risk/critical), 14-day activity histogram, intervention pipeline, health distribution, recent activity, recent feedback. Read-only.
- **`GET/POST /api/v2/feedback`** — GET admin-only (list). POST any user `{ message ≤4000 }` → inserts `platform_feedback`; if the table is missing and email is configured, falls back to emailing `PLATFORM_ADMIN_EMAIL`.
- **`POST /api/v2/demo-seed`** — Body `{ teamId?, reset?=true }`. Wipes team-scoped rows (order matters: children before `goals` due to self-FK), then plants 1 goal, 6 field objects, 7 change_events, 3 assumptions, 7 confidence signals, 3 interventions, and updates team_spaces intent/tensions. Owners are fictional (Priya, Marco, Sana, Diego, Arun) — they leak into AI answers if not cleaned.
- **`POST /api/v2/simulate-teammate`** — Body `{ teamId?, scenarioIndex? }`. Picks one of 5 hard-coded teammate statements, runs it through interpret+apply.

---

## 6. Client — `app/page.tsx` (the entire UI)

One `"use client"` module (~4400 lines). Root layout (`app/layout.tsx`) loads Inter / Newsreader / JetBrains Mono and `globals.css`; no providers.

### 6.1 Design system (`globals.css`)

CSS variables on `:root`: warm paper palette (`--bg:#f7f5f0`, `--ink:#141413`, `--muted:#7a7770`, `--surface:#ffffff`), accents `--accent:#1e4d3b` (forest green), `--warn:#a64234` (rust, blockers/tension), `--warm:#b37a28` (ochre, reminders), `--cool:#325c99` (requests), `--shift:#5c4b93` (shifts). Radii `10/16px`, three shadows, three font tokens. Buttons: bordered surface default, `.btn-primary` (ink fill), `.btn-ghost`. Many named classes (`.shell`, `.topbar`, `.hero`, `.section`, `.commitment` + per-type colors, `.conf-bar`, `.timeline-item` + accent variants, `.voice-orb` + `.recording` pulse, `.sheet`, `.rich-reply`, `.ask-*`, `.auth-*`, `.just-updated` 2.2s flash). Responsive breakpoints at 720px and 480px.

### 6.2 `Page` — root state & flows

**State:** `session`, `checkingAuth`, `state: LatticeState` (init `emptyLatticeState`), `tab` (`pulse|timeline|interventions|commitments`), `orbKick` (counter to trigger voice), `chatPrefill`, `loading`, `teams`, `activeTeamId`, `needsTeam`, `teamPanel` (`none|create|manage`), `feedbackOpen`, `adminOpen`, `members`, `simulating/analyzing/seeding`, `liveStatus`, `isPlatformAdmin`, `updatedIds: Set<string>`. Ref `loadStateRequest` (monotonic, discards stale `/state` responses).

**`markUpdated(ids)`**: adds ids to `updatedIds`, sets `liveStatus="change just landed"`, then after **2400ms** clears them and resets to `"watching"` (drives the realtime flash).

**Auth:** `supabase.auth.getSession()` on mount + `onAuthStateChange` (clears state to empty on sign-out). Sign-in/up happens in `AuthGate` via `signInWithPassword` / `signUp({ options:{ data:{ name } } })`.

**`authedFetch(input, init)`** (the critical wrapper): throws if no session; sets `Authorization: Bearer <access_token>`; sets `Content-Type: application/json` **only if** there is a body, no existing content-type, and the body is NOT `FormData/Blob/URLSearchParams`. This is what lets voice upload multipart audio with the browser's boundary intact (the original bug: forcing JSON on FormData broke transcription).

**Teams:** `loadTeams()` (GET `/api/v2/teams`; empty → `needsTeam`); `loadState(teamId?)` (GET `/api/v2/state`, guarded by `loadStateRequest` so fast team-switching never shows stale data); effects load roster into `members`. Member identity is matched by **display name (case-insensitive)** everywhere (reassign, `ownedByMe`, `canMutate`) — so `name` is load-bearing and `MyProfile` refuses to save it empty.

### 6.3 Render gating

`checkingAuth` → "Loading…" → `!session` → `<AuthGate>` → `needsTeam` → `<FirstTeamGate>` → main `.shell`: `Topbar`, `Tabs`, the active view, floating `VoiceDock`, conditional modals.

### 6.4 Realtime

`useEffect` creates channel **`lattice-${activeTeamId}`** with four `postgres_changes` listeners (`event:"*"`, `schema:"public"`, filter `team_space_id=eq.<id>`) on **`change_events`, `interventions`, `goals`, `field_objects`**. A shared handler grabs the changed row id, calls `markUpdated([id])` (flash), then `loadState(activeTeamId)` (full re-pull, not incremental).

### 6.5 Tabs & views

**Pulse** (`PulseView`, default): goal hero (title from active goal or `state.intent`, `ConfidenceSparkline`, "Give an update" → orb kick, "Edit goal" → `GoalEditor` → `POST /api/v2/goal`) · `StatusBar` (at-risk + blocker dots; label all clear / needs attention / watching) · `MyProfile` · `MorningBrief` · `Nudges` · `LatticeChat` · "What changed" (≤5 `TimelineItem`) · "What to do next" (top-2 interventions, read-only) · "Open tensions" (first 3).

**Timeline** (`TimelineView`): "Plan vs reality" two-column (struck-through previous goal + drifting commitments | active goal + aligned/drifting counts) via `goalDrift`; full change log as detailed `TimelineItem`s (shows `detail` + `impact.teamReadable`); "Patterns worth noticing" from `structuralAnalysis` when >1 blocker.

**Interventions** (`InterventionsView`): suggested (sorted urgency desc) as `InterventionCard` (Dismiss → `dismissed`, Mark acted → `acted`, urgency pill High/Med/Low, `.urgent` ≥4) + a dimmed "Already handled" list. PATCHes `/api/v2/intervention`.

**Commitments** (`CommitmentsView`): field objects grouped by type in fixed order `[promise, blocker, request, reminder, shift, signal]`, sorted within a group by due urgency (closed last via sentinel `1e15`, undated `1e14`, else `Date.parse(dueAt)`); plus an assumptions section. Each `CommitmentRow` shows type chip (tooltip `typeDescription`), owner reassign button, "assigned to you", status, due meta (warn if late), deferred/declined notes, confidence (`round(conf*100)% conf` + `.conf-bar`). Actions gated on `!closed && canMutate(f)` (owners/admins always; members only their own items): **Done** (promise/request → `complete`), **Resolved** (blocker → `resolve`), **Set due** (`DuePicker` → `set_due`), **Can't do** (`RespondPopover` → `defer`/`scope_change`/`decline` + reason), **Drop**. Owner popover lists `members` (name/email/role) + Unassign → `set_owner`.

### 6.6 The unified chat (`LatticeChat`)

The single surface for ask + tell. `MAX_TURNS = 6`. Sample chips: `["What's a signal vs a shift?", "What am I forgetting?", "Blocked on auth — need Priya today.", "Who's overloaded?", "How does this app work?"]`.

**Client-side intent classification — `looksLikeQuestion(text)`:**
1. ends with `?` → question.
2. else first word (lowercased) ∈ `{what, who, when, where, why, how, which, is, are, am, do, does, did, can, could, should, would, will, was, were, have, has}` → question.
3. else → an update (tell).

**Routing:** question → `POST /api/v2/ask` `{ query, teamId, history }` → renders `data.answer`. update → `POST /api/v2/interpret` `{ input, apply:true, teamId }` → `onState(data.state)` applies immediately (no preview step in the live chat) + renders `interpretation.reply` plus up to 3 `richReply.recorded` bullets. Both read `res.text()` first then JSON-parse defensively so HTML error pages surface as readable diagnostics.

**Voice:** mic → `getUserMedia({audio:true})` → `MediaRecorder`; on stop, picks extension from `recorder.mimeType` (Safari → `audio/mp4`), builds `FormData("audio", blob, "update.<ext>")`, `POST /api/transcribe` via `authedFetch`; on success feeds `data.text` into `send()` (re-classified).

**Orb integration:** `orbKick` effect scrolls chat into view, focuses input, starts recording. `prefill` effect (from a nudge "Reply") drops text into the input. The floating `VoiceDock` orb and the hero "Give an update" both bump `orbKick` (switching to Pulse first).

### 6.7 Other components

`MyProfile` (inline editable name/skills/focus/bio, save-on-blur → profile PATCH, shows `statsForMember` summary) · `MorningBrief`/`BriefColumn` · `Nudges` (Reply → prefill chat, Snooze → client-only) · `TypeInfoButton` (i popover) · `ManageTeamModal` (members, roles, remove, invites, join-link `${origin}/join/${joinToken ?? id}`, join-requests, email digest) · `FeedbackModal` (submit; admins also list) · `AdminDashboardModal` (KPIs, activity bar SVG, health/intervention bars, watchlist, feedback) · `CreateTeamModal`/`FirstTeamGate` (create team or paste invite token). `ComposerSheet` is defined but **not rendered** (legacy preview→apply sheet replaced by `LatticeChat`).

### 6.8 Public pages

- **`/invite/[token]`**: unwraps token, shows sign-in/up tabs; once authed, auto-`POST /api/v2/invitations/accept` and `router.replace("/")` on success.
- **`/join/[token]`**: `GET /api/v2/join-links/[token]` to show team name; once authed, auto-`POST /api/v2/join-requests`; "Request sent" or, if already a member, redirect home.

Both attach the bearer token manually (they predate/don't use `authedFetch`).

---

## 7. Derived-on-read logic (pure functions, no storage)

### `lib/v2.ts`
- `teamConfidence(state)`: no goal & no commitments → `0.7`. Else `goalC = activeGoal.confidence ?? 0.7`; no commitments → `goalC`; else `round((goalC*0.5 + mean(commitmentConfidences)*0.5), 2)`.
- `atRiskCount(state)`: promises with `confidence < 0.5` OR id present in any blocker's `links`.
- `goalDrift(state)`: keywords = active goal title words **length > 3**; a promise is **aligned** if it has a link starting `intent-` OR any keyword substring-matches `"<title> <detail> <status>"`; else drifting.
- `structuralAnalysis(state)`: over blockers — `overloaded` = owners with **≥2** blockers; `recurring` = title tokens (length **≥4**) appearing **≥2** times; `totalBlockers`.
- `formatRelative(iso)`: `<1m` "just now", `<60m` "Nm ago", `<24h` "Nh ago", else "Nd ago".
- Presentation maps `labelForChangeKind`, `glyphForChangeKind`, `accentForChangeKind` (good/warn/shift/muted).

### `lib/nudges.ts` — `deriveNudges(state)`
Thresholds: **stale commitment 72h**, **open blocker 24h**, **stale assumption 168h (7d)**. `lastTouch` = latest change_event per `target_id`. `isDeferred` skips anything with `deferred_until` in the future. Five kinds: `overdue_commitment` (promise past `due_at`, urgency 3 if ≥48h late else 2), `stale_commitment` (no touch ≥72h, urgency 3 if ≥144h), `open_blocker` (no touch ≥24h, urgency 3 if ≥72h), `stale_assumption` (≥168h since last check, urgency 3 if `at_risk` else 1), `overdue_reminder` (trigger text looks past). Sort: urgency desc, then ageHours desc. Route returns first 6.

### `lib/member-stats.ts` — `statsForMember(state, name)`
Matches owner by normalized name. Over the member's promises/requests: `completed` (status done/resolved), `openCount`, `overdueCount` (open + `due_at` past), `declinedCount` (dropped with `decline_reason`), `onTimeRate` (done-on-time / done-with-due, null if none), `avgDeliveryHours` (mean of completion−firstTouch hours, rounded, null if none). `firstTouch` = earliest change_event per target. `summarizeStats` renders `"N shipped · N open · N overdue · N% on-time · ~Nh avg"`.

---

## 8. AI layer

### `lib/persona.ts`
- **`LATTICE_PERSONA`**: a seasoned chief-of-staff voice. Lead with the answer; name people/numbers/dates; contractions; dry/wry; 2–4 sentences; willing to say what's missing and push back. Hard bans: "Sure", "Great question", "as an AI", "based on the provided data", hedging, emoji, exclamation points, all-caps.
- **`APP_KNOWLEDGE`**: canonical one-line definition + one example for each primitive, plus a crib of how each tab/feature works and the confidence-vs-progress rule. Injected as a separate system message in `/api/v2/ask`.

Both are shared constants so the voice never diverges across interpret/ask/brief (and the fallback strings).

### `lib/ai-v2.ts` — `interpretV2(input, state, members)` (primary path)
System prompt = `LATTICE_PERSONA` + (1) role framing, (2) the **exact JSON schema** Lattice must return: `{ reply, richReply{headline,recorded[],implications[],suggested[]}, entities[], changes[], goalShift?, assumptions?[], interventions?[], followUpQuestion?, broadcast?[], confidenceImpact? }`, (3) principles (blocked/waiting → blocker+blocker_emerged; finished/done → commitment_completed; scope/deadline detection; reuse over duplicate), (4) **due-date extraction** (resolve "by Friday"/"EOD"/"tomorrow" to absolute ISO 8601 UTC; EOD=23:59, morning=09:00; never invent), (5) **owner-extraction rules** (critical): name a person → put ONLY that name in `owner` and strip assignment phrasing from `title`; "I/me/my/self" is NOT an assignment → leave owner empty; never default to the reporter; one owner per entity, emit multiple entities for multi-assignment; with a `TEAM_MEMBERS` block, *suggest* owners by skills/load/history in `reply`/`suggested` but never auto-assign.

**Context block** (`buildContextBlock`): active goal + confidence; `TEAM_MEMBERS` (name, role, skills, focus, summarized delivery stats); last **10** commitments; first **8** assumptions; **5** most recent change_events. Model `OPENAI_MODEL ?? gpt-5.4-mini`, `response_format json_object`, no temperature. Falls back to `fallbackInterpretV2` (regex keyword parser) on missing key / empty / invalid JSON.

### `lib/ai.ts` — legacy V1 `interpretWithOpenAI` + `lib/lattice.ts` `fallbackInterpretation`
V1 interpreter retained mainly for the deterministic fallback used when OpenAI is unavailable.

---

## 9. Build order to recreate from scratch

1. **Supabase project.** Create it; enable `pgcrypto`. Apply, in order: all enums (§3.1) → V1 tables (team_spaces, team_members, field_objects, memory_events, delegated_requests, reminders, interpretations) → V2 tables (goals, change_events, assumptions, dependencies, confidence_signals, interventions) → team_invitations → platform_feedback → team_join_requests → the added columns (`field_objects.due_at/deferred_until/decline_reason`, `team_members.skills/focus/bio`, `team_spaces.created_by/join_token`). Then functions, triggers, indexes, RLS policies (§3.3–3.4), realtime publication (§3.5). **Drop** `on_auth_user_created_add_orgmind_member` after creating it.
2. **Next.js app.** `next@16` App Router, React 19, TS strict, Tailwind v4 PostCSS. Add `@/` alias to root.
3. **`lib/`**: `supabase.ts` (4 client factories: browser-memoized, service-role, server, user-server-with-bearer) → `auth-server.ts` (`requireUserSupabaseClient`) → domain types (`lattice.ts`, `v2.ts`) → persistence (`team-state-db.ts`, `v2-db.ts`) → derivations (`member-stats.ts`, `nudges.ts`) → AI (`persona.ts`, `ai-v2.ts`, `ai.ts`) → `teams.ts`, `email.ts`, `task-digest.ts`, `platform-admin.ts`.
4. **API routes** (§5). Every protected route starts with `requireUserSupabaseClient`; service-role only where listed.
5. **Client** (`app/page.tsx`, `layout.tsx`, `globals.css`, public pages) per §6.
6. **Verify:** `npx tsc --noEmit`, `npx eslint app lib`, `npm run dev`, sign up, create a team, type "Priya is blocked on auth, demo's Friday", confirm a blocker + a Friday-due commitment + a change_event land and the Timeline/Nudges reflect them.

---

## 10. Known quirks / invariants to preserve

- **Confidence is not progress.** Always label it `conf`.
- **Never default owner to the reporter.** "I/me/self" leaves owner empty.
- **Owner matching is by display name, case-insensitive, string-based.** Renaming a member does not rewrite owner strings on existing rows — history is history; reassignment is the fix.
- **Destructive actions (drop/reassign/defer/decline) are owner-gated** in both UI (`canMutate`) and server (commitment owner-gate). `set_confidence` is the one ungated action.
- **Deferred items don't nudge.** `deferred_until > now()` skips every nudge kind.
- **Derive on read before you store.** Nudges, brief, member stats — pure functions; no second table, no cron.
- **One voice everywhere** via shared `LATTICE_PERSONA`.
- **`authedFetch` must not force `application/json` on FormData** — or voice breaks.
- **Stale-response guard** (`loadStateRequest`) on `/state` is required for fast team switching.
- **Branding drift:** persona/UI say "Lattice"; the task-digest email CTA and feedback email subject still say "OrgMind"/"Orgmind" (the project's former name). Decide whether to unify.
- **Demo-seed plants fictional owners** (Priya/Diego/etc.) that leak into AI answers until cleaned.
