# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

An internal **client onboarding dashboard** for NP Accel (NPA). It helps the
onboarding team run the full client onboarding lifecycle — visualizing the
workflow, collecting platform access from clients, and identifying automation
opportunities to shorten time-to-value.

Two deployed pages:

- **`index.html`** — the team-facing dashboard (login required)
- **`portal.html`** — the client-facing access setup guide (opened via a unique per-client link)

## Architecture

- **Single-file, no build step.** Each page is one self-contained HTML file with
  inline CSS and JS. No framework, no bundler, no `npm install`. Edit the HTML
  directly and reload.
- **Firebase** (project `npa-onboarding`) provides:
  - **Auth** (email/password) — gates `index.html`
  - **Realtime Database** — live cross-user sync
- Firebase config is inline near the top of each file. It is a client-side
  public config (safe to expose); security is enforced by Auth + Realtime
  Database rules.

### `index.html` — team dashboard

Five tabs, switched by `switchTab(name)`:

1. **Onboarding Board** — data-driven swimlane board (SEO / Paid / Email) with
   phases, cards, drag-and-drop, inline editing, RACI, links, notes, and status.
   Rendered from `DEFAULT_PHASES` / `DEFAULT_SWIMLANES` / `DEFAULT_CARDS`.
   Board state persists at Firebase `/board`.
2. **Miro Reference** — embedded Miro board (NPA Process R&R).
3. **Access Tracker** (`atRender`, `at*` functions) — per-client platform access
   status. Clients stored at Firebase `/access_clients`, keyed by client id.
   Each client has a generated `key` used to secure their portal link. AMs pick
   which platforms a client uses via a grouped checklist (`AT_PLATFORM_GROUPS`).
   Generates the unique portal URL and follow-up emails.
4. **Onboarding SOP** (`sopRender`, `sop*` functions) — the full 2026 onboarding
   checklist (`SOP_DATA`, 7 stages) with each step rated `high` / `partial` /
   `manual` for automation potential, plus automation-opportunity themes
   (`SOP_THEMES`). Analysis-only; no persistence.
5. **Access Project** (`apRender`, `ap*` functions) — live tracker for the
   Blubird Systems access pilot (`AP_STEPS`/`AP_PHASES`, 30 steps across 7
   phases, Closed Won → Welcome Call). Baseline status per step; managers click a
   status badge to cycle it. Status overrides persist at Firebase
   `/access_project` as `{stepId: status}` and sync live. Statuses:
   `done` / `in-progress` / `blocked` / `to-confirm` / `not-started`.

**Permission model** (`ROLES`): viewer / editor / manager / admin. Enforced
client-side (UI gated on `perms()`), with Realtime Database rules as backstop.

### `portal.html` — client access guide

- Opened via `portal.html?client=<id>&key=<key>`. The key is validated against
  the client's Firebase record before any content shows; a bad/missing key shows
  an "access denied" screen.
- Reads the client's `platforms` array and renders **only** the relevant platform
  cards (Google, Paid, Website/CMS, Email/ESP). Each card has step-by-step access
  instructions, the correct NPA email(s), and the required role.
- Client clicks "Mark as Done" per platform → writes to the client's `completed`
  array in Firebase → the team's Access Tracker updates live.

## Canonical platform names

Platform identifiers **must match exactly** between `index.html`
(`AT_PLATFORM_GROUPS`) and `portal.html` (`data-platform` attributes +
`markDone()` calls). They are the keys used for completion tracking. Website/CMS
is a single platform (`Website`) in the tracker; the client picks their specific
CMS inside the portal.

## NPA access emails (in portal instructions)

- **Google / paid / CMS platforms** → `analytics2@npaccel.com` (plus specific
  Google-team emails per platform: torik@, kelseyh@, destiny.martin@, etc.)
- **Email / ESP platforms** → `kim.macomber@npaccel.com`

## Deployment

- **`origin`** = `kwmoore22/npa-onboarding` (public). Hosts the **live GitHub
  Pages site** at https://kwmoore22.github.io/npa-onboarding/. Push here to
  update the live app: `git push origin`.
- **`org`** = `npa-automation/npa-onboarding` (private). The org's working home.
  `git push` (default) targets this.

Portal/access links are currently hardcoded to the `kwmoore22.github.io` Pages
URL (`AT_PORTAL_URL` in `index.html`). If the hosting URL changes, update it there.

## Conventions

- Keep everything in the single HTML files — do not introduce a build system or
  split into modules unless explicitly asked.
- Match the existing inline style: prefixed function names per feature
  (`at*` for access tracker, `sop*` for the SOP tab), `esc()` for HTML escaping.
- After editing, sanity-check JS with:
  `node --check <(sed -n '/<script>$/,/<\/script>/p' index.html | grep -v '<script\|</script')`
- Commit only when asked. Push to `origin` when the change should go live.
