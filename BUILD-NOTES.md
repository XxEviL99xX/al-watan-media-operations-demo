# Media Operations demo — build notes

**Cloned from** the v6 demo (baseline commit `65d93da`) on 2026-07-22. That v6 demo is now
archived at `demos/_archive/workflow-demo/` — do not edit it.
**Remote:** ⚠️ **`origin` → `XxEviL99xX/al-watan-media-operations-demo` (PUBLIC).**
**Pushing to it publishes the live site**, https://xxevil99xx.github.io/al-watan-media-operations-demo/ .
(An earlier version of this note said the remote was empty. It is not — correct as of 2026-07-23.)
**Storage key:** `watan_mediaops_v7` (its own, so it never inherits v6 localStorage).

Single self-contained file: `index.html`. No build step. Open it directly.

---

## 1. Al Watan ERP theme

Replaced the demo's navy palette with the ERP design tokens: official red `#B32025`,
light neutral surfaces, IBM Plex Sans Arabic, semantic status colours (brand red is
**not** used for every state). Dark mode re-tuned — the old dark block still carried
navy. Both themes verified.

## 2. ERP shell header

`AL WATAN ERP` mark · 9-dot **All Modules** · `ERP / Media Operations` breadcrumb ·
spacer · notifications · `ع / EN` · light/dark switch · profile · logout icon.

- **No production role switcher** (blueprint §114). Sign out → login screen has
  one-click demo accounts.
- **Logout is its own icon** (one click, `aria-label`, RTL-mirrored). Clicking the
  name does nothing — identity only.
- **9-dot menu** opens the wider ERP module list (Home, Dashboard, Requests,
  Calendar, Media Operations, Production, Digital Assets, Approvals, Reports,
  Clients/CRM, Vendors, Settings). Only **Media Operations** is active; the rest are
  deliberately disabled — they prove the shared ERP without faking screens.
- Brand and breadcrumb click back to the queue.

## 3. v7 workflow tickets

| Ticket | What changed |
|---|---|
| **T1** | Coverage type + event type default to `— اختر —` and are required. Root cause was `viewNew()` defaulting new requests to `comprehensive`/`editorial`. Client auto-fill was *already* identity-only; the hint now says so. Client fields stay **editable** after auto-fill. |
| **T2** | `تعديل الطلب` for the requester while status is `marketing_approval`; disappears once Marketing approves. Reuses the existing edit path so the before/after diff is logged automatically. Separate heading, button label and notification distinguish it from resubmit-after-changes. |
| **T3** | `needsEditorial()` no longer counts `hebaNeeded` — that was why a named person still routed to the Editorial Manager. Named people are assigned on submit; the Editorial Manager panel shows them as already assigned with no picker. **Mixed rule works**: Heba + generic journalist → still routes for the journalist, Heba never re-picked. |
| **T4** | `إنهاء المرحلة` deleted from Shooting/Editing. `finishStageIfDone()` advances when the **last** assigned member stamps `إنهاء العمل`. Preparation keeps one explicit action — see OPEN below. |
| **T6** | `ready_to_publish` owner moved off `["editor","client"]` to `["publisher"]`. Dropped the "set caption/channel" guidance that contradicted the no-URL decision. **Revised 2026-07-22: approval is a clean handover — see below.** |

### T6 revision — the client's queue empties on approval (2026-07-22)

⚠️ **This supersedes the earlier "one final-change round from Ready to Publish"
decision** (recorded as user-confirmed) and blueprint §1160, which kept a secondary
client window open after approval.

The client's *only* decision point is **نسخة العميل**. There they may approve, request
changes, or reject. The moment they approve:

- status → `ready_to_publish` and the request **disappears from the client's list**
  (`ready_to_publish` is deliberately absent from the client's visible statuses);
- the client has **no action anywhere** on it;
- it lands with the publisher, who gets the ⚡ flag and the Publish action;
- after publish it reappears for the client under **Archive**, read-only.

Cancellation and change requests must therefore happen **before** approval.

Two implementation notes worth keeping:

- The access guard runs in `render()` **before** it dispatches to the detail view, not
  inside `viewDetail()`. Returning different HTML from `viewDetail()` still left
  `bindDetail()` wiring up controls that no longer existed, which threw.
- `doAction`'s `cl_final` branch is now **unreachable** and is kept only in case Ahmed
  reinstates the one-round window. It is commented as such.

## 4. Dashboard

Stat cards (Total / In production / Due this week / Completed), then two columns:
queue left, rail right (**Upcoming Actions**, **Your action queue**, role context).

- **Upcoming Actions** is an *agenda*, not a to-do list — every active request you can
  see, soonest deadline first, top 3, with ⚡ on the ones you personally owe. Built
  as a to-do list first and rendered empty for the Requester, which is why it changed.
  No "View Calendar" — there is no calendar in this system.
- **Archive and Cancelled are filters, not nav tabs.** No separate Archive page;
  `state.view="archive"` redirects into the filter. Blueprint §303 order:
  `All · Needs your action · Awaiting approval · In production · With client · Archive · Cancelled`.

## 5. Assign team — modal wizard

Opens from the **queue row or the detail page**, so the Head never has to enter the
request. Two steps with a numbered progress bar:

1. **Select the people** — Shooting + Editing chips; editorial shown read-only with
   per-selection state (`يُحال إلى مسؤول المحررين` vs `مُعيَّن مسبقًا`).
2. **Set the work order** — ONE unified drag-reorderable sequence across *all*
   assigned people (this was the fix: it previously only listed shooters, so anyone
   picked from other teams vanished). ▲▼ is the required non-drag alternative
   (blueprint §747). Below it, a horizontal **team timeline** preview showing who
   goes 1st, 2nd, 3rd with their role.

The chosen order is written to the audit trail as a numbered work order. The fixed
stage strip (Shooting → Editing → Review → Client → Published) is shown as
non-reorderable.

## 6. Request detail

Two columns. Record left; rail right with **Workflow guide** (You are here → Next
step → Do now, green when it's your action), **Production team members** (numbered
when >1), **Internal review**. Rail is hidden entirely in client view so the client
never sees internal people or review groups.

## 7. Bug fixes found while building

- **Work was possible during a pending change request.** The brief could still
  change, yet staff could assign/start/review/publish against it. Now frozen for
  everyone except Marketing (decide) and the Requester (withdraw) — action buttons
  *and* the start/finish tracking buttons.
- **Assign audit only logged the shooters.** Now logs the full ordered team.
- **Responsive was silently dead.** Media queries sat *before* the base rules, so at
  equal specificity the base rules won. All responsive CSS now lives at the end of
  the stylesheet — see the comment there. Header holds one row from 1400px to 375px
  by shedding label → glyphs → breadcrumb prefix → role line → switcher → name →
  breadcrumb.
- **Status badges looked like buttons** (solid, same green as the action beside
  them). Now tinted pills with a coloured dot. Also `assign_team` status reads
  **"بانتظار التعيين" / "Awaiting assignment"** — a state, distinct from the
  "تعيين الفريق" action.
- **"Waiting on" listed four reviewer names.** Now names the group(s):
  `مجموعة المراجعة ١ / ٢`.
- **Restart-at-In-progress offered for internal reviewer changes.** `revisionSource`
  now distinguishes `review` (→ Shooting/Editing only) from `client` (→ full re-plan).
- **Note modal had no attachments.** `askNote` now carries files through
  `doAction → addTl → timeline`, rendered via the shared `mediaGrid`, and the
  storage-quota fallback strips timeline file URLs too.
- Optional **backup email** added, matching the backup-phone pattern (wired through
  the form, diff and audit).
- **New request had no Back button.**

---

## ✅ CLOSED since this note was first written (2026-07-23)

1. ~~**Who closes Preparation.**~~ **Answered by the user:** Preparation shows a single
   **▶ بدء العمل**, and pressing it advances Preparation → Shooting *and* stamps the
   presser's location in the same click (commits `db26a2c`, `c71ba33`).
2. ~~**Who presses نشر.**~~ **Answered by Ahmed:** the `media_manager` role. The
   `publisher` placeholder account **"جهة النشر (قيد التأكيد)" was deleted** and نشر was
   folded into the **Production Head** (commit `e8ded0f`). The "Publishing Officer /
   مسؤول النشر" (`aabdulla`) collision is resolved — he stays a stage-2 *reviewer* and
   has nothing to do with publishing.
   ⚠️ **One thing still to confirm with Ahmed:** which human role `media_manager` maps
   to. Working assumption = **Production Head** (built that way). If he says Marketing
   Manager, it is a one-line owner/gate swap.

## ⚠️ Still OPEN — needs Ahmed

1. **At التصوير, who belongs in the بانتظار list?** It currently shows the shooting crew
   **plus** the editorial staff (مراسل / صحفي / هبة), per the rule that they are present
   at the shoot. The user said "I should only go to فريق التصوير" but no change was made
   — narrowing it is a workflow-rule change, so **ask before changing**.
2. **Does the client keep a final-change round after approving?** The demo says no —
   approval is a clean handover (see T6 revision above). That reverses an earlier
   confirmed decision, so it is worth putting back in front of Ahmed.

## Not done yet

- Client review portal still uses the internal shell; blueprint P14 wants a separate
  minimal branded portal.
- Blueprint P08/P11–P18 screens (Editorial queue, review workspace, revision routing,
  cancellation, ready-to-publish, archive, cancelled) still use the generic detail page.
- **Blueprint conflict to settle:** P04 §567 says approval should happen *inside the
  detail workspace* and warns against one-click approval in a dense row. The queue
  now does inline quick actions because that was explicitly requested. By the
  blueprint's own priority table (§40) the latest feedback wins — but
  `frontend-page-blueprint.md` should be updated to match, or this will regenerate.
- **Blueprint self-contradiction:** P01 §290–310 specifies both three work indicators
  (Needs action / Due this week / With client) *and* a filter row containing "Needs
  your action" and "With client". They duplicate.
