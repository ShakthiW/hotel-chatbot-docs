# Multi-Tenant Provisioning & Landing-Page Theming

How a new hotel gets a live admin dashboard and (optionally) a public marketing website, how
platform-wide code updates reach every tenant safely, and how a landing-page theme gets
installed, swapped, or updated across the fleet — end to end, across five repositories.

---

## 1. The Repositories Involved

| Repo | Role |
| :--- | :--- |
| `Standord-AI/chatbot-demo-api` | The Go backend. Owns the `Property` record, including which landing-page theme (if any) a hotel has, and exposes the admin endpoints that trigger the pipelines below. |
| `Standord-AI/chatbot-demo-admin` | The template repo. Every tenant's `standord-deploy/<slug>-admin` repo starts life as a copy of this. |
| `Standord-AI/hotel-landing-themes` | A **library** of landing-page themes (not a hotel site itself) — `page.tsx` + `components/` + `theme.css` per theme, hotel-agnostic, installed into a tenant repo by copying. |
| `Standord-AI/hotel-chatbot-onboarding` | GitHub Actions pipeline that provisions a brand-new hotel tenant end to end. |
| `Standord-AI/hotel-chatbot-rollout-pipeline` | GitHub Actions pipelines that push updates to *already-onboarded* tenants — either a core-app template update (`rollout.py`) or a landing-page theme update (`theme_rollout.py`). |
| `Standord-AI/chatbot-super-admin-dashboard` | The internal tool platform staff use to trigger all of the above without touching `curl` or GitHub Actions directly. |

Two independent GitHub Actions pipelines exist because they touch **disjoint sets of files** in
a tenant repo (see §2) and run on different triggers — onboarding runs once per hotel;
rollout runs whenever the platform team wants to push an update to hotels that already exist.

---

## 2. Three-Way File Ownership Split

Every tenant repo (`standord-deploy/<slug>-admin`) is partitioned into three ownership zones.
Each automated pipeline is only ever allowed to touch its own zone — this is what makes it
safe to push a core-app update without wiping a hotel's custom landing page, or push a theme
update without touching a hotel's own content.

| Zone | Files | Owned / synced by | Never touched by |
| :--- | :--- | :--- | :--- |
| **Core app** | Everything else (dashboard pages, chat UI, API routes, `src/lib/agent/prompts/*` is separately protected too) | `hotel-chatbot-rollout-pipeline/scripts/rollout.py`, from `chatbot-demo-admin@main` | `theme_rollout.py` |
| **Theme** | `src/app/page.tsx`, `src/components/landing/**`, `src/app/theme.css` (if the installed theme has one) | `hotel-chatbot-rollout-pipeline/scripts/theme_rollout.py` (or `hotel-chatbot-onboarding/scripts/install_theme.py` at signup), from `hotel-landing-themes@<theme_ref>` | `rollout.py` — these three paths are in its `PROTECTED_TENANT_FILES` list, so a core-app rollout backs them up, overlays the fresh template, then restores them untouched |
| **Hotel content** | `src/website/content.ts` | Nobody automated — hand-edited per hotel, or (later) a dashboard editor | Both pipelines — also in `rollout.py`'s `PROTECTED_TENANT_FILES`, and `theme_rollout.py` never lists it as a target path at all |

A hotel with no theme installed simply has no `src/app/theme.css` and its `src/app/page.tsx` /
`src/components/landing/` are still holaa's own default marketing page (the same files
`chatbot-demo-admin` ships) — installing a theme for the first time is exactly the same
operation as updating one later.

---

## 3. Pipeline: Hotel Onboarding (`hotel-chatbot-onboarding`)

Provisions a brand-new tenant, triggered via `repository_dispatch`/`workflow_dispatch` (see
that repo's README for the full `curl` payload and required secrets). In order:

1. Registers the `Property` + `hotel_owner` `User` against `chatbot-demo-api`, sets plan/token
   limit/add-ons.
2. Provisions a Qdrant collection and a LangSmith project.
3. Creates `standord-deploy/<slug>-admin` from a tarball of `chatbot-demo-admin@main` and pushes.
4. **If an optional `website_theme` + `website_theme_ref` input is given**, runs the same
   overlay logic as `theme_rollout.py` (`scripts/install_theme.py`) on the just-created repo —
   `page.tsx`, `components/landing/`, and `theme.css` (if present) — before the next step.
   Left blank, the hotel keeps holaa's default marketing page, same as today.
5. Links the repo to Vercel and deploys to production (`vercel --prod`), injecting the
   dynamic env vars (`NEXT_PUBLIC_MERCHANT_UUID`, `QDRANT_COLLECTION`, etc.).

`install_theme.py` is deliberately a simpler, single-repo sibling of `theme_rollout.py` rather
than a shared dependency — these are two separate repos and the codebase already doesn't share
code across that boundary (`onboard.py` and `rollout.py` don't either).

---

## 4. Pipeline: Core-App Template Rollout (`rollout.py`)

Pushes the latest `chatbot-demo-admin` template to one, several (not currently supported — see
§6 for why theme rollout got this and template rollout didn't yet), or every tenant repo.

- Clones the tenant repo, backs up every path in `PROTECTED_TENANT_FILES` (tenant prompt/config
  files, `.env.local`/`.env.production`, and the three theme/content paths from §2) if present,
  wipes everything except `.git`, overlays the fresh template, then restores the backed-up
  paths over it.
- Commits and pushes to `main` (no `--force`; a rejected push means something changed the repo
  between the clone and this push and the run fails loudly rather than guessing).
- Confirms the push actually reached production via a real Vercel deploy call
  (`vercel link` / `vercel --prod`) rather than assuming the push alone triggers one.
- Aborts the remaining repos in a run if the **first 3** processed all fail their Vercel
  deploy — a systemically broken template update shows up in the first few hotels, not all of
  them.

See `hotel-chatbot-rollout-pipeline/README.md` for the exact `curl`/dashboard trigger payload
and required secrets (shared with theme rollout, §5).

---

## 5. Pipeline: Landing Theme Rollout (`theme_rollout.py`)

Installs or updates one theme from `hotel-landing-themes` on one or more tenant repos.
Mirrors `rollout.py` in reverse: instead of preserve-these-paths/overlay-everything-else, it's
**overlay-only-these-paths** (`page.tsx`, `components/landing/`, `theme.css`) and touches
nothing else in the repo — never `src/website/content.ts`, never any core-app file.

`theme.css` is handled as present-or-absent: if the theme being installed has one it's copied
to `src/app/theme.css`; if it doesn't, any existing `theme.css` on the target repo is deleted —
so switching between a themed and a theme.css-less theme never leaves a stale file behind.

Downloads the theme from a **pinned ref** (a tag or commit SHA, never `main`/"latest") so an
already-installed theme never silently drifts until a rollout is deliberately triggered again.

### Targeting

`TARGET_REPO` accepts three shapes:

| Value | Behavior |
| :--- | :--- |
| `ALL` (or empty) | Discovers and installs on **every** tenant `-admin` repo, regardless of what theme (or none) each one currently has. Use for a fresh-install-everywhere push, not for updating one theme's users — see the warning below. |
| A single repo name | Installs on just that repo. |
| A comma-separated list of repo names | Installs on exactly those repos, in order. Each name is validated the same way a single repo is — one invalid name fails the whole run rather than silently skipping it. This is what the super-admin dashboard sends when you pick "target hotels currently on theme X" (§8). |

**Important:** `ALL` here means literally every tenant, independent of which theme is
installed where. There is intentionally no "discover everyone on theme X" mode inside this
script — that resolution happens one layer up, in the dashboard/backend (§7–§8), which reads
`Property.WebsiteTheme` and passes back an explicit repo list. Running `THEME=theme-1
TARGET_REPO=ALL` would install theme-1 on a hotel currently running theme-2 or theme-3 too.

Same consecutive-failure abort (3) and Vercel-deploy verification as `rollout.py`.

---

## 6. The Landing-Page Themes Library (`hotel-landing-themes`)

A themes **library**, not a hotel site — locally it also doubles as a Next.js app so a theme
can be previewed, type-checked, and built, but only one theme's files are ever "installed"
(copied into `src/app/` / `src/components/landing/`) at a time, matching how a real tenant
repo only ever has one theme installed.

```
hotel-landing-themes/
  themes/
    theme-1/            # "Warm Editorial" — warm sand/teal/terracotta, Fraunces serif
      page.tsx
      theme.css
      components/*.tsx
    theme-2/             # "Bold Monochrome" — stark black/white/gold, sharp corners, bold sans
      page.tsx
      theme.css
      components/*.tsx
    theme-3/             # "Panoramic" — warm off-white/black + single yellow accent, Playfair Display serif
      page.tsx
      theme.css
      components/*.tsx
  src/
    app/
      page.tsx           # = a copy of whichever theme was last previewed
      theme.css           # = that theme's theme.css (or absent, if it has none)
      globals.css          # core-app-owned, neutral fallback only — never carries theme colors
    components/landing/    # = that theme's components/
    website/
      content.ts            # hotel-owned static content contract (see below)
      lib/data.ts            # shared dynamic-data contract (see below)
  scripts/preview-theme.mjs # local preview harness — mirrors theme_rollout.py's copy exactly
  tsconfig.json              # excludes raw themes/ from project-wide type-checking
```

### Previewing a theme locally

```bash
pnpm preview theme-2   # copies themes/theme-2/{page.tsx,theme.css,components/} into src/
pnpm dev                # http://localhost:3000
```

`scripts/preview-theme.mjs` performs the **exact same copy operation** the real pipeline
scripts do, so a local preview is a faithful test of the actual install mechanism, not a
parallel dev-only path. `tsconfig.json` excludes `themes/` from `tsc`'s project-wide scan
(only one theme's imports resolve at a time via `src/components/landing/`); verifying a
non-active theme means previewing it first, then type-checking/building the resulting `src/`.

### The shared dynamic-data contract (`src/website/lib/data.ts`)

Every theme fetches hotel data through these functions — never hits the backend directly —
so any theme in the library works against any hotel without modification:

| Export | Backend source |
| :--- | :--- |
| `getPropertyProfile()` | `GET /api/v1/properties/{id}` |
| `getRoomTypes()` | `GET /api/v1/properties/{id}/rooms` (active rooms only) |
| `getOutlets()` | `GET /api/v1/properties/{id}/outlets` |
| `getAttractions()` | `GET /api/v1/properties/{id}/attractions` |
| `getMedia()` / `getFirstVideo()` / `getPhotos(limit?)` | `GET /api/v1/properties/{id}/media` |

All public GET endpoints (no auth), resolved via `NEXT_PUBLIC_MERCHANT_UUID` /
`NEXT_PUBLIC_PROPERTY_API_URL`, wrapped so an unreachable backend degrades to empty/null
rather than throwing — every theme renders a graceful "being added — check back shortly"
empty state instead of crashing.

### Hotel-owned static content (`src/website/content.ts`)

For the handful of things with no backend model — because they're inherently curated, not
aggregated — themes read an optional, hand-edited `WebsiteContent`: `tagline`, `aboutUs`,
`story`, `contact` fallback, `testimonials?: Testimonial[]`, `ownerLetter?: OwnerLetter`. There
is deliberately no fabricated review-count or fake menu-item data anywhere in the theme
library — where a reference design has a feature with no real backing data (a "12 Reviews"
counter, an itemized restaurant menu, a live-availability search), the theme either uses real
data through a different, honest component (e.g. a tabbed dining panel over real
`PropertyOutlet`s instead of invented dishes) or omits the feature outright.

### Adding a new theme

1. `themes/<slug>/page.tsx` + `themes/<slug>/components/**` + `themes/<slug>/theme.css`, using
   absolute imports (`@/components/landing/...`, `@/website/...`) — never relative, since the
   pipeline relocates `page.tsx` two directory levels up from `components/` on install.
2. `theme.css` defines the shadcn color tokens (`--background`, `--primary`, `--accent`, …)
   plus `--font-heading`; `globals.css` stays a colorless fallback so there is exactly one
   source of truth per theme.
3. `pnpm preview <slug>` → `tsc --noEmit` → `pnpm build` → visually check every section in a
   browser with the backend unreachable (confirms empty states) before considering it done.

---

## 7. Backend Tracking (`chatbot-demo-api`)

`Property` carries which theme (if any) a hotel has installed:

```go
WebsiteTheme        string  `json:"website_theme"`          // empty = no theme, default holaa page
WebsiteThemeVersion string  `json:"website_theme_version"`  // a tag/commit in hotel-landing-themes, never "main"
```

- `PUT /api/v1/admin/properties/{id}/website-theme` (`SetWebsiteTheme`) — super-admin only.
  **Only records intent** — it does not push any files itself; actually installing/updating
  the theme is a separate `theme-rollout` pipeline trigger. Same split as plan assignment
  (`SetPlan`) vs. template rollout.
- `POST /api/v1/admin/pipelines/theme-rollout/trigger` (`TriggerThemeRollout`) — dispatches
  `hotel-chatbot-rollout-pipeline`'s `theme-rollout.yml` with `target_repo` (single name,
  comma-list, or `ALL`), `theme`, `theme_ref`. Forwards `target_repo` through unchanged — it
  does **not** resolve `ALL` against `WebsiteTheme` itself (see §5's warning); the dashboard
  resolves the repo list client-side from `GET /api/v1/properties` before calling this.
- `GET /api/v1/properties` (super-admin) already returns the full `Property` record for every
  tenant, `website_theme`/`website_theme_version`/`slug` included — no separate listing
  endpoint was needed to support the dashboard feature in §8.

---

## 8. Super-Admin Dashboard Controls (`chatbot-super-admin-dashboard`)

- **Hotels list** (`/`) — a **Theme** column shows each hotel's `website_theme` at a glance
  (a `—` badge when none is installed).
- **Hotel detail** (`/hotels/{id}`) — a "Website theme" field calls `SetWebsiteTheme` directly,
  same pattern as the plan dropdown.
- **Pipelines → Landing theme** (`/pipelines`) —
  - **"Hotels currently on each theme"**: groups every property with a non-empty
    `website_theme` by theme slug, shows the hotel count and names, and a **"Target these N"**
    button per group. Clicking it fills Target repo with `<slug>-admin,<slug>-admin,...` for
    exactly that group and pre-fills the theme slug — leaving just the new version to fill in.
  - The trigger form underneath still accepts manual input (single repo / explicit list /
    `ALL`) for a fresh install onto a hotel with no theme yet.
  - Both call `POST /api/v1/admin/pipelines/theme-rollout/trigger` and poll the run via
    `usePipelinePolling("theme_rollout", correlationId)`, same as the other two pipeline tabs.

---

## 9. Common Workflows

**Onboard a new hotel with a theme already installed** — trigger `hotel-chatbot-onboarding`
with `website_theme`/`website_theme_ref` set; no separate theme-rollout trigger needed
afterward.

**Push an edited theme to every hotel currently running it** — Pipelines → Landing theme →
find the theme's group under "Hotels currently on each theme" → **Target these N** → fill in
the new `theme_ref` (a fresh tag/commit — never re-use "main") → Trigger theme rollout. Only
those hotels are touched; everyone on a different theme (or none) is untouched.

**Add a theme to a hotel that has none** — Pipelines → Landing theme → type the hotel's
`<slug>-admin` repo name manually, pick the theme and a pinned `theme_ref` → trigger. Also
sets `SetWebsiteTheme` on the hotel detail page afterward so the dashboard's tracking reflects
reality (the rollout trigger and the DB record are two separate, deliberately decoupled
actions — see §7).

**Roll out a core-app fix without touching any hotel's theme or content** — Pipelines →
Rollout template — completely separate from the above; `PROTECTED_TENANT_FILES` already
guarantees `page.tsx` / `components/landing/` / `theme.css` / `content.ts` survive untouched.
