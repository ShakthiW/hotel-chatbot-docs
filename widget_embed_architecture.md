# Embeddable Widget Architecture (`public/widget.js`)

`chatbot-demo-admin/public/widget.js` is the script a real hotel actually embeds on its own
website — a `<script>` tag loaded cross-origin from an arbitrary domain the platform doesn't
control. It is **not** the same thing as the admin dashboard's Live Preview (`ChatSimulator.tsx`)
or the in-room QR guest chat (`GuestChatWidget.tsx`) — those are React, share
`WidgetRenderer.tsx`, and get every new card type automatically. `widget.js` is a completely
separate, hand-written vanilla-JS renderer for the same `ui_payload` contract (see
`chat_gateway_api.md`), because a `<script>` tag embedded on someone else's site can't ship a
React runtime. **Every new payload type has to be added here by hand, or the guest widget
silently shows only the AI's prose with no card at all** — this has been the source of real
parity gaps (see §4).

---

## 1. How a Hotel Embeds It

```html
<script
  src="https://your-deployment.example.com/widget.js"
  data-property-id="a8360f9a-445f-405a-9725-232113f382b4"
  data-api-url="https://your-deployment.example.com"
></script>
```

`data-api-url` should be an absolute URL to wherever `chatbot-demo-admin` is deployed
(Vercel, typically) — **not** the Go backend's own URL. The widget talks to the Next.js app's
`/api/chat` and `/api/v1/*` rewrite proxy (see §3), the same way the admin dashboard itself
does; it never has its own separate API base pointed at the Go backend directly. Omitting
`data-api-url` falls back to `http://localhost:8080` with a loud console warning — silently
talking to localhost on a real hotel's site would otherwise be invisible until someone noticed
the widget was dead.

`chatbot-demo-admin/public/demo.html` is a working reference embed, useful for testing
`widget.js` changes directly (rather than through `ChatSimulator.tsx`, which does **not**
exercise this file at all).

---

## 2. Structure

The entire file is one top-level IIFE — `(function () { ... })()` — with a version guard
(`window.__HOTEL_CHATBOT_LOADED__`) so loading the script twice on the same page is a no-op.
Everything (styles, DOM construction, the SSE client, all rendering functions) lives in this
one function's closure; there's no module system or bundler involved. `WIDGET_VERSION` (in the
file's own header comment) should be bumped on any behavior-affecting change — it's the only
way to tell which version a given page loaded, surfaced via
`window.__HOTEL_CHATBOT_VERSION__` and a startup console log.

Key internal pieces:
- **`escapeHtml(str)`** — HTML-escapes admin-controlled config text and all AI/RAG-derived
  content before it reaches `innerHTML`. Every card-rendering function is expected to route
  every interpolated field through this — see §5.
- **`safeImageUrl(url)`** — only allows `http(s)://` URLs through into an `href`/`src`,
  blocking `javascript:`/`data:` scheme injection via a compromised logo or booking URL.
- **`renderHTMLCarousel(payload)`** — the central dispatcher, one `if/else if` chain keyed on
  `payload.type`, mirroring `WidgetRenderer.tsx`'s dispatch on the React side.
- **`renderComparisonTable(rooms)`** — a pure client-side render triggered by a room card's own
  Compare button; no round-trip to the AI (see §6).

---

## 3. Why Fetches Use Relative Paths, Not an Absolute API URL

`next.config.ts` on `chatbot-demo-admin` rewrites any relative `/api/v1/:path*` request to the
real Go backend server-side:
```ts
async rewrites() {
  return [{ source: "/api/v1/:path*", destination: `${process.env.NEXT_PUBLIC_PROPERTY_API_URL}/api/v1/:path*` }];
}
```
Every reliable call in this codebase — including `widget.js`'s own polling and card-rendering
data — goes through this relative-path proxy. A **direct, absolute-URL fetch straight to the
droplet from the browser** (bypassing this proxy) was the actual root cause of a real,
hard-to-diagnose bug: `ChatSimulator.tsx`'s session-history hydration used
`` `${apiBase}/api/v1/...` `` instead of a relative path, and reproducibly failed with a bare
`TypeError: Failed to fetch` specifically on a cold page load — while the exact same request
typed manually in the console, or re-triggered without a full reload, always succeeded. Fixed
by switching to the relative-path pattern everywhere a browser-side fetch was using an absolute
URL instead. **New code added to any of the three chat surfaces should default to a relative
`/api/v1/...` path and only use an absolute URL (`apiUrl`/`apiBase`) when there's a specific
reason the request can't go through this proxy** — WebSockets are the one real exception, since
`next.config.ts` rewrites are HTTP-only and don't proxy the WS upgrade handshake.

---

## 4. Rendering Parity Matrix

All 14 `ui_payload` types (see `chat_gateway_api.md` for the full catalog) are now handled —
this table exists to make the *next* gap visible immediately when a new payload type is added
to the tools layer, rather than silently missing from this file the way five of them once were.

| `type` | widget.js | Notes |
| :--- | :---: | :--- |
| `rooms` | ✅ | Filter chips, real scarcity badges, "Matches: ..." reasoning tag, room image, Explore/Compare quick actions |
| `no_rooms_found` | ✅ | "Explore all rooms" resends the unfiltered request |
| `outlets` | ✅ | |
| `dining_menu` | ✅ | |
| `spa_treatment` | ✅ | |
| `attractions` | ✅ | Google Maps link when available, "Learn More" quick action otherwise |
| `weather` | ✅ | |
| `personal_recommendation` | ✅ | |
| `preferences` | ✅ | Fully interactive — click-to-toggle chips, Save composes and sends a summary |
| `booking_hold` | ✅ | |
| `payment_link` | ✅ | |
| `itinerary` | ✅ | |
| `experience_timeline` | ✅ | |
| `human_handover` | ✅ | Static status card — does not yet live-update in place the way `ChatSimulator.tsx`'s does via WebSocket `state_change` events |
| `booking_confirmation` | ✅ (dead) | Rendering branch exists but no tool currently emits this type — kept for forward compatibility, not evidence anything is broken |

When adding a new tool payload type: add the render branch here, add the row above, and check
whether the field names match what the tool actually emits (`tools/index.ts`) — several past
bugs in this file were exactly that mismatch, not a rendering logic error.

---

## 5. Security Model

- **XSS**: every interpolated field in a card template must go through `escapeHtml()` before
  reaching `innerHTML`. This matters specifically because card content (room descriptions,
  outlet details, etc.) traces back to tool output influenced by ingested hotel documents via
  RAG — not fixed, trusted, admin-authored copy. A real gap here (the card renderer was the one
  path skipping this) was found and fixed; see the file's own header comment for the policy.
- **URL injection**: any field rendered into an `href` or `src` (booking links, checkout URLs,
  room images, Google Maps links) goes through `safeImageUrl()` first, rejecting anything that
  isn't a plain `http(s)://` URL.
- **Multi-tenancy**: `widget.js` itself just forwards whatever `propertyId` the embedding
  `<script>` tag declares — the actual tenant-isolation guarantee lives server-side, in the
  LangGraph tools trusting `configurable.propertyId` over their own optional argument (see
  `chat_gateway_api.md`).

---

## 6. Interactive Features Implemented in Vanilla JS

Two Phase 3 features needed genuine DOM-manipulation logic here, not just a template addition:

**Room comparison** — each room card embeds the *entire* room array for its carousel as an
HTML-escaped JSON string in a `data-rooms` attribute (`e(JSON.stringify(payload.rooms))`). The
Compare button's click handler parses it back out and appends a fresh comparison-table bubble
directly via `messagesContainer.appendChild(...)` — no AI round-trip, capped at 4 rooms since a
table wider than that stops being readable in a docked chat panel regardless of layout.

**Live re-ranking** — two module-level variables, `lastCarouselEl` and `lastCarouselType`,
track the most recently rendered carousel. When a new `rooms`/`no_rooms_found` payload arrives
and the *previous* carousel was also one of those two types, the previous carousel element is
removed from the DOM before the new one is appended — collapsing a search refinement into one
current result instead of letting stale results pile up. An unrelated card type in between
(dining, attractions) leaves an older room card alone, since that isn't a refinement. Both the
filter-chip row and the carousel itself are wrapped in one `.hcb-carousel-wrapper` element
specifically so this removal takes both out together — they're sibling elements in the raw
HTML string, not one nested inside the other, so removing only the carousel would have
orphaned the chip row above it.

---

## 7. Accessibility

The chat launcher is a `<div>` with a click handler, which by itself is invisible to keyboard
navigation — a real, previously-unaddressed gap, not a hypothetical one. Current state:
- `role="button"`, `tabindex="0"`, `aria-label`, and a `keydown` handler (Enter/Space) on the
  launcher, plus `aria-expanded` kept in sync and focus moved into the conversation on open.
- `aria-label`s on every icon-only control (close, fullscreen, send).
- `role="log" aria-live="polite" aria-atomic="false"` on the message list, declared in the
  initial HTML (not injected later) — matched in `ChatSimulator.tsx` and `GuestChatWidget.tsx`
  for consistency across all three surfaces.
- Visible `:focus-visible` outlines on every interactive control, and `prefers-reduced-motion`
  support disabling transitions/animations.

**Known gap, not yet addressed**: the live region does not currently mute itself during
token-by-token streaming, so a screen reader may announce intermediate partial text rather than
only the finished message. Muting during stream and announcing once on completion would be the
next real improvement here, not a cosmetic one.
