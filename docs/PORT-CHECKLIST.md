# Lattice V2 — Port Checklist (V1 → V2 client)

> What the V2 rewrite still has to bring forward from V1.
>
> **Diagnosis.** The backend is intact. Every API route and `lib/` file is byte-for-byte
> identical between V1 (`../lattice/lattice`) and V2 — verified by line count across all 40+
> files. V2 *added* the event core (`capture`/`entities`/`undo` routes, `events.ts`/`fold.ts`/
> `view.ts`). The entire gap is in **one file**: `app/page.tsx` went **4,402 → 858 lines**. The
> old page is preserved verbatim at `app/legacy-v1/page.tsx` (confirmed identical to V1).
>
> The new page built only the **capture → entities → 3-lens (mine/team/missing)** core and never
> rebuilt the ~25 other components. It also has **no Topbar/menu chrome**, so the dropped surfaces
> are unreachable even where their APIs are live.
>
> **This is a UI port, not a backend rebuild.** For every ❌ item below, the backing API already
> exists and is unused. Graft V1's modals/views onto the new event-core + lens model. Per the V2
> PRD (§5.7, §6.5), most of these are marked **"(Ported.)"** — required, not cut for clutter.

Legend: `[ ]` todo · ❌ gone · ⚠️ partial (some of it survives in the new page).
Line numbers reference `app/legacy-v1/page.tsx` unless noted.

---

## Phase 0 — Navigation chrome (prerequisite)

Nothing below is reachable without this.

- [ ] **Topbar / app menu** — port `Topbar` (L468–597). Exposes: team switcher, Manage members &
  invites, Profile, Feedback, Admin (admin-only), Give-an-update. The new page has no nav at all.

---

## Phase 1 — Unblock the basics (P0)

### 1.1 Personal profile ❌ — `MyProfile` (L2371–2621)
- **Why first:** the new page tells the user *"Set your name in the team so work can be routed to
  you"* but provides **no way to set it** — a functional dead-end.
- **API (exists):** `PATCH /api/v2/teams/{teamId}/members/profile`
- **Fields:** `name` (req, ≤60), `skills[]` (≤20 × ≤40 chars, comma-input → array), `focus` (≤200),
  `bio` (≤600). Save on blur/Enter; idle→saving→saved(1.6s)→error.
- **Stats block:** shipped / open / overdue / on-time % via `lib/member-stats.ts` `statsForMember()`.
- **Done when:** a member can set their display name and see it reflected in the "mine" lens routing.

### 1.2 Create team — post-create invite ⚠️ — `CreateTeamModal` (L710) / `FirstTeamGate` (L597)
- **Now:** `TeamGate` + `POST /api/v2/teams` works but is name-only and dead-ends on
  "you can invite people after."
- [ ] Wire the "after": surface invite link + email invite immediately post-create.

### 1.3 Invites & join links ❌ — invite section of `ManageTeamModal` (L805–1176)
- **Email invite.** API `POST /api/v2/teams/{teamId}/invites` `{ email, role }` → copyable
  `/invite/{token}` (page exists, 180 lines). Auto-joins on `POST /api/v2/invitations/accept`.
  Revoke: `DELETE …/invites?inviteId=`.
- **Shareable join link.** Show `/join/{token}` (page exists, 244 lines) →
  `GET /api/v2/join-links/{token}` → stranger submits `POST /api/v2/join-requests` `{ token, message? }`.
- **Done when:** an owner/admin can copy both link types and an invited user lands in the team.

### 1.4 Manage team + roles ❌ — `ManageTeamModal` (L805–1176)
- **APIs (exist):** `GET/PATCH/DELETE /api/v2/teams/{teamId}/members`,
  `PATCH /api/v2/teams/{teamId}/join-requests`.
- [ ] Member list with profiles (name/email/role/skills/focus/bio).
- [ ] Change role (owner/admin/member); only **owner** may promote to owner.
- [ ] Remove member (`DELETE …/members?memberId=`).
- [ ] Pending invites list + revoke.
- [ ] Join-request review: approve / reject (`PATCH …/join-requests {joinRequestId, action}`).
- **Role matrix:** owner = full; admin = invite/manage/approve but no owner-promote/-demote;
  member = self-profile + own items only. (New page already gates *mutation* via `canMutate()`.)

---

## Phase 2 — "What's Slipping" lens (the founder posture; PRD §5.4, §6.4)

These belong in the existing `MissingLens`, not as separate dashboards.

- [ ] **Morning Brief** ❌ — `MorningBrief`/`BriefColumn` (L2706–2816). `POST /api/v2/brief {teamId}` →
  `{ changed[], atRisk[], needsDecision[], generatedAt }`, ≤3 bullets each, Refresh button.
- [ ] **Nudges** ❌ — `Nudges` (L2816–2920). `GET /api/v2/nudges?team=` → `{ nudges[] }`
  (`{prompt, person, reason, urgency 1–3}`); Reply (prefill capture) + Snooze.
- [ ] **Goals / goal-drift** ❌ — `PulseView`/`GoalEditor` (L1204–1440). Team active goal + confidence
  sparkline; set/edit via `POST /api/v2/goal`.

---

## Phase 3 — Admin & Feedback (gating + APIs already done; pure UI)

### 3.1 Feedback ❌ — `FeedbackModal` (L4252–4402)
- **API (exists):** `POST /api/v2/feedback {message}` → `platform_feedback` table (falls back to
  emailing `PLATFORM_ADMIN_EMAIL`). Admins also `GET /api/v2/feedback` (403 otherwise).
- [ ] Message textarea (≤4000) for all users.
- [ ] Admin-only "All submissions" triage list (email + timestamp + message + Refresh).

### 3.2 Admin dashboard ❌ — `AdminDashboardModal` (L3856–4252)
- **Gating:** `isPlatformAdminEmail()` (`lib/platform-admin.ts`); button admin-only;
  `GET /api/v2/admin/overview` 403s otherwise.
- [ ] 6 KPI cards (teams / people / updates-14d / open-work / health / invites).
- [ ] 14-day activity bar chart.
- [ ] Health-mix + intervention-pipeline stacked bars.
- [ ] Team watchlist: per-team health score 0–100 (`computeTeamHealth`), worst-first, 6 metrics each.
- [ ] Recent platform activity + feedback inbox.

---

## Phase 4 — Maturing the surface

- [ ] **Ask / Q&A chat** ❌ — `LatticeChat` (L2920–3255). Routes questions (`?`/Q-words) →
  `POST /api/v2/ask`; updates → `POST /api/v2/interpret`. Voice via `/api/transcribe`. (Could fold
  into the existing Composer.)
- [ ] **Preview-before-apply** ❌ — `ComposerSheet`+`RichReply` (L3417–3658). `interpret` with
  `apply:false` → structured preview (headline / recorded / implications / suggested / follow-up)
  before committing. New Composer applies directly with no preview.
- [ ] **Interventions** ❌ — `InterventionsView` (L1569–1685). suggested/acted/dismissed + urgency;
  `PATCH /api/v2/intervention {id, state}`.
- [ ] **Timeline** ❌ — `TimelineView` (L1440–1569). Plan-vs-reality, full change log, patterns
  (overloaded members, recurring tokens).
- [ ] **"Can't do" with reason** ⚠️ — `RespondPopover` (L2272). New page hardcodes
  `reason:"plan changed"` on defer; restore decline/defer/scope_change + free-text reason
  (`/api/v2/commitment`).
- [ ] **Assumptions** ❌ — at-risk/invalidated section (`/api/v2/assumption`).
- [ ] **Status bar** ❌ — `StatusBar`/`StatusDot` (L2621–2706): at-risk/blocker chips + qualitative
  label. Minor; can fold into a lens header.
- [ ] **Onboarding / cold-start** ❌ — `demo-seed` + `simulate-teammate` routes unused. PRD §6.5
  flags cold-start as a *new* requirement; decide seed-team vs guided first-run.

---

## Already ported (no action)

✅ Auth (email/password + Google OAuth) · ✅ Create/switch team · ✅ Text + voice capture
(`/api/v2/capture`, `/api/transcribe`) · ✅ Entities + mine/team/missing lenses (`deriveView`) ·
✅ Realtime snapshot patching · ✅ Per-event **Undo** (`/api/v2/undo`) · ✅ Role-gated mutation
(`canMutate()`) · ⚠️ Commitment actions (done/resolve/set-owner/set-due/defer/drop survive in
`EntityCard` via the new `/api/v2/entity` path).
