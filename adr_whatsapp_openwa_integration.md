# ADR: Connecting Hotel WhatsApp Numbers via OpenWA

> **Doc**: ADR · WhatsApp-01 &nbsp;·&nbsp; **Revision**: 7 &nbsp;·&nbsp; **Revised**: 2026-09-21
> **Status**: Phases 1–4 shipped (core) · Social Inbox next &nbsp;·&nbsp; **Scope**: Guest concierge channel

The guest-facing concierge — reachable by a direct message or an Instagram/Facebook ad
click-through — is live: text and image replies verified against real inbound traffic on a real
connected number, with the round trip closed end to end.

> This is a markdown export of the canonical, illustrated ADR artifact. The rendered version
> (with the two architecture diagrams) is the source of truth going forward; this file exists so
> the decision record lives in the repo alongside the code it describes. See
> `whatsapp_integration_api.md` in this folder for the implementation reference this ADR informed.

---

## §01 — Context

Guests reach the concierge through exactly one channel today: the web widget, talking to a
LangGraph agent (`POST /api/chat` on `chatbot-demo-admin`) over Server-Sent Events. Phases 1–3
(§13) shipped a real, monetized staff-facing WhatsApp channel — but a hotel's WhatsApp number
still couldn't hold a conversation with a guest at all before this work. This ADR designs (and,
as of Revision 7, largely ships) the actual guest-facing concierge, for the two ways a real
guest ends up in that conversation — **messaging the hotel's number directly**, or **tapping a
Click-to-WhatsApp ad on Instagram or Facebook** that opens WhatsApp with a prefilled message to
that same number.

Three things grounded the original design, verified directly rather than assumed:

- **OpenWA supports real inbound webhooks** — per-session, HMAC-signed, at-least-once with
  idempotency keys and retry-with-backoff, plus a documented delivery-failure log and a polling
  fallback. The gateway side of "a guest messages the hotel" was a solved problem, not a gap.
- **Click-to-WhatsApp ads work mechanically with any number, official or not — but attribution
  doesn't.** The ad-to-WhatsApp deep link is just a prefilled draft message; it works identically
  regardless of whether the destination number runs the official Cloud API or OpenWA. Meta's
  `referral` metadata (which ad, which creative, `ctwa_clid`) is a Cloud-API-only enrichment —
  confirmed absent from OpenWA's webhook payload entirely, across every doc and its issue
  tracker. §09 designs around this honestly.
- **The web agent's capability surface is bigger and messier than assumed** — a full read of
  `chatbot-demo-admin/src/lib/agent/` found 17 real tools (2 never bound to any subagent) and 19
  generative-UI payload types (4 of which no tool actually emits). §07 is the corrected inventory.

OpenWA ([rmyndharis/OpenWA](https://github.com/rmyndharis/OpenWA)) remains the gateway. Everything
in this ADR is additive to what Phases 1–3 already ship — the staff relay, the entitlement gate,
the recipient model, the capacity monitor all stay exactly as they are.

---

## §02 — Decision Drivers

| Driver | Why it matters |
| :--- | :--- |
| Time to first hotel live | Some hotels want this in weeks, not after a Meta Business verification cycle. |
| Cost per conversation | Cloud API bills per conversation past the free tier; a self-hosted gateway doesn't. |
| Account-safety exposure | An unofficial client carries real, non-zero ban risk — genuinely two-way traffic, not just outbound staff alerts. |
| Reuse over rebuild | The web agent's tools, RAG, entitlements, and memory already exist — the guest-facing WhatsApp agent should be a composition layer over them, not a parallel agent. |
| Guest acquisition via ads | A hotel running Click-to-WhatsApp ads needs the concierge to actually answer — attribution reporting is a separate, honestly-scoped concern (§09). |
| Identity that outlives a cache clear | The web widget's guest identity lives in `localStorage` — gone on a new device or a cleared cache. A phone number doesn't have that failure mode. |
| Exit cost later | If a hotel later needs the official API, swapping the gateway shouldn't mean rebuilding the agent. |

---

## §03 — Options Considered

| Option | Time to live | Cost | Account-safety | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **A · OpenWA, self-hosted (chosen)** | Days | Infra only | Real, mitigable | Unofficial client (`whatsapp-web.js` / `baileys`). Real inbound webhooks confirmed (§05). No ad-attribution metadata (§09). |
| B · Meta WhatsApp Cloud API, direct | Weeks | Per-conversation | None (official) | Business verification per hotel; template-approval flow; real `referral` ad-attribution object. |
| C · Cloud API via a BSP (Twilio / 360dialog / Gupshup) | 1–2 weeks | Per-conversation + markup | None (official) | Faster onboarding than B, still official, still gets ad attribution; adds a billing relationship per hotel. |
| D · Do nothing (web widget only) | — | None | None | Leaves both acquisition paths with nothing on the other end. |

---

## §04 — Decision

**Shipped (Phases 1–3): OpenWA, self-hosted, as a one-way staff notification relay.** Every
hotel connects a number itself, from Integrations → WhatsApp; a guest's service request is
matched against that property's `WhatsAppRecipient` rows and relayed to whichever match, gated
by the `whatsapp_enabled` entitlement, with usage-quota billing and automated capacity
monitoring on top.

**Shipped now (Phase 4): the same number receives, too.** A guest — organic or ad-driven —
reaches a WhatsApp-native composition of the exact same agent (tools, RAG, subagent routing,
entitlements) that already serves the web widget, not a second agent built from scratch. As of
this revision, the full round trip is live and verified against real inbound WhatsApp traffic on
a real connected number — guest identity resolution, text replies, and image composition for the
room/dining/spa payload types all confirmed working end to end, not just designed. §13 has the
exact shipped/remaining split.

> **Correction, caught in review.** The entitlement check originally only gated the
> connect/recipients management endpoints — not the send trigger inside `CreateServiceRequest`
> itself. A hotel whose add-on lapsed would have been locked out of the settings UI while still
> getting notifications sent for free, indefinitely. Fixed by checking
> `entitlements.Bool(db, propertyID, "whatsapp_enabled")` inline in the send path itself —
> verified live.

> "I cannot say the risk is zero … the project recommends several safeguards such as rate
> limiting, realistic delays between messages, avoiding bulk messaging to unknown numbers."
> — OpenWA maintainer, in response to a community ban-risk question (issue #60)

Still a genuine trade-off: hotels opt in explicitly, dedicate a number, and are shown OpenWA's
own safety guidance. Two-way traffic raises the stakes slightly — an inbound flood is now a real
scenario, not just outbound volume the hotel itself controls; see the risk register (§12).

### Why a dedicated composition layer, not the web agent's raw output

Routing WhatsApp through the existing `/api/chat` SSE route and flattening whatever came back
into plain text would work, but throws away exactly what would make the channel feel native: a
generative-UI room-card payload becomes a wall of text instead of an actual photo; there's no way
to offer a real, tappable choice; replies read like a transcript, not a text conversation. A
distinct composition step — same tool library, same RAG, same entitlements, different
turn-composition — lets the model decide up front "I'm sending three room photos," not "I'm
rendering a React component that then needs translating." §08 has the corrected, verified version
of what that composition step can and can't reach for.

### Watching the one thing that can actually fall over

OpenWA runs as a single shared container holding every connected hotel's session. It has no
`/health` or `/metrics` endpoint, but self-reports memory via `GET /api/sessions/stats/overview`;
a ticker job (`platform-controller/capacity_monitor.go`, 15-minute interval) polls that, compares
reported RSS against the container's configured cap, and raises a deployment-wide `PlatformAlert`
at 70%/85%. Surfaced on the super-admin Platform Health page, with a documented,
escalation-ordered runbook (`docs/openwa-capacity-runbook.md`). Turning on inbound traffic makes
this more load-bearing, not less — a guest-facing channel means the session actually has to stay
`ready`, not just idle.

---

## §05 — Inbound Architecture: How a Guest's Message Actually Arrives

**Shipped and verified live.** OpenWA's webhook system is real and fairly complete
(`docs/06-api-specification.md` §6.4.8, §6.6) — per-session registration, HMAC-signed delivery,
at-least-once with idempotency keys and retry-with-backoff, a delivery-failure log, and two
independent polling fallbacks. The gateway side was never the risk; wiring it into the existing
shared-core agent was the actual work, and the whole chain has now sent and received real
messages against a real connected number.

### The three-hop round trip, and why it stays this shape

OpenWA is never called directly by the frontend, only ever through `chatbot-demo-api` — one Go
service owns every OpenWA credential and every session. The guest-facing agent's actual
intelligence (tools, RAG, subagent routing, memory) lives where it already lives, in
`chatbot-demo-admin`'s LangGraph graph — reused, not reimplemented.

1. **OpenWA → chatbot-demo-api.** A `message.received` webhook, registered once per property at
   connect-time, delivers to `POST /api/v1/integrations/whatsapp/webhook/{sessionId}`.
   HMAC-verified against a per-session secret generated at registration. `sessionId`
   reverse-resolves to the owning `Property`.
2. **chatbot-demo-api → chatbot-demo-admin.** The verified payload forwards to
   `POST /api/messaging/inbound` — channel-generic, carrying `channel: "whatsapp"` — gated by
   `X-Internal-Secret`. Fire-and-forget from the Go side: an LLM round trip easily exceeds
   OpenWA's 10s webhook timeout, so the receiver acknowledges OpenWA immediately and lets the
   agent turn run in its own goroutine.
3. **chatbot-demo-admin runs the graph.** The same `hotelAgent` graph the web widget uses,
   invoked non-streaming with `configurable: {thread_id, propertyId}` — `thread_id` derived from
   the guest's JID (§06).
4. **The WhatsApp composer turns the graph's output into send instructions.** Scans every
   `ToolMessage` in the trajectory for `__UI__` payloads, maps the image-bearing types to real
   `send-image` items with captions, appends the AI's own closing text (§08).
5. **chatbot-demo-admin → chatbot-demo-api → OpenWA → guest.** The composed item list posts to
   `POST /api/v1/properties/{id}/integrations/messaging/send`, dispatched to `pkg/openwa`'s
   `SendText`/`SendImage`, strictly in order, synchronously. `SendLocation`/`SendContact`/
   `SendPoll`/`SetTyping` remain unbuilt.

This is two Go↔Next.js hops for one reply — the same trade every internal-secret-gated guest-flow
call in this codebase already makes, and it's what keeps OpenWA's credentials in exactly one
place.

### Per-thread serialization — shipped, closing a real race

A guest double-texting fires two separate webhook deliveries; without a guard, both independently
invoke `hotelAgent.invoke()` against the exact same LangGraph `thread_id` concurrently, with no
ordering guarantee over which finishes last and overwrites the other's saved checkpoint. The web
widget avoids this by disabling its input box mid-stream — WhatsApp has no equivalent. Fixed with
a per-`(property, phone)` mutex in `chatbot-demo-api` (`threadLocks`): a second message arriving
mid-turn blocks until the first's full round trip has actually finished.

### Inbound rate-limiting — shipped

A fixed per-JID cap, enforced at the webhook receiver before a message ever reaches the graph: a
sender JID beyond 10 messages in a 5-minute sliding window (both configurable) gets a single
canned "please try again later" reply and nothing further reaches the agent for the rest of that
flood; past 20, messages are silently dropped entirely. v1 only — a heuristic/adaptive layer is
explicitly future scope (§14).

### Message-kind filtering — shipped, after a real incident

Originally filtered only on OpenWA's `isGroup` boolean. A real WhatsApp Channel broadcast (a
news/business channel the connected number happened to be subscribed to, not a guest) arrived
with `isGroup: false` and was accepted — creating a fake `GuestProfile`, a fake `ChatSession`,
and forwarding the broadcast text to the agent, which failed trying to send a reply back to a
channel JID. OpenWA's spec says `kind` (`individual|group|channel|status|broadcast|unknown`) is
the authoritative discriminator that supersedes `isGroup` — the receiver now accepts only
`kind === "individual"`. Caught from real inbound traffic, not code review — see §12.

### Human handover — the AI-mute is shipped; the Social Inbox itself is not

**Shipped:** when `request_human_handover` fires with a `"connecting"` outcome (a
live-handover-capable property), `chatbot-demo-admin` sends that turn's own acknowledgment
reply, then flips the session's `session_mode` to `handover_requested` via the existing
`request-handover` endpoint. From that point, the webhook receiver checks the session's mode
before ever forwarding to the agent — a message arriving while the mode isn't `ai_active` is
persisted but never reaches the LLM. Deliberately **not** applied when the tool returns
`"logged"` (a Starter/Discover property with no live-handover entitlement) — those properties
have no way to ever call `handback` (gated behind the same entitlement), so muting the AI there
would strand the guest permanently.

**Not yet built:** anything for a staff member to actually see or join a muted WhatsApp
conversation. `TakeoverSession`/`HandbackSession` already exist and work (staff-JWT +
`liveHandoverEnabled` gated) — `handback` is what un-mutes a session today, but the only way to
call it is a direct authenticated API call, not a UI action. The plan stays a new, dedicated
**Social Inbox** page, separate from `/dashboard/conversations` (which is built around
web-widget-specific assumptions — a sub-50ms WebSocket, "guest is on the page right now" — that
don't fit WhatsApp's asynchronous nature). This is the single biggest remaining piece of Phase 4.

What it will reuse once built: the `ChatSession`/`ChatMessageRecord` data model (`channel:
"whatsapp"` is populated on every real message now), the `TakeoverSession`/`HandbackSession`
handlers, and `request_human_handover`'s existing entitlement check. The extensibility point that
matters for Messenger/Instagram DM later is the channel-generic normalization already shipped
into the inbound/outbound endpoints — adding a channel means one new webhook receiver and one new
sender behind the existing switch, not a new inbox or data model.

---

## §06 — Identity, Sessions & Continuity

The web widget's guest identity (`guestId`) lives in the browser's `localStorage` — gone on a new
device or cleared cache, phone stored only as an attribute, never the key. WhatsApp doesn't have
that failure mode: the guest's phone JID *is* a stable, durable identity. This is the one place
WhatsApp is a plain upgrade over the web widget, not just parity with it.

- **`thread_id` — the phone JID, scoped per property.** LangGraph's `MemorySaver` checkpoints per
  `thread_id`, deterministic from `propertyId` + guest JID, so the same guest resumes the same
  checkpointed conversation automatically.
- **Identity is property-scoped.** The same phone number messaging two different hotels is two
  completely separate, isolated guest profiles — never merged.
- **Cross-channel linkage — resolved, simpler than it looked.** Dedicated research (grepping
  every `models.GuestProfile` constructor) found there is no populated web-side profile to link
  WhatsApp identity *to* — `GuestProfile`'s only writer, `BindSessionToGuest`, has a frontend
  caller (`/api/memory/bind`) with zero call sites in the actual widget.
- **So the design inverts — and it's shipped.** WhatsApp became `GuestProfile`'s first real
  writer: `FindOrCreateGuestByPhone(propertyID, phone)`, called from the inbound webhook receiver
  before the message reaches the graph. Verified against real inbound messages. Guarded by a
  DB-level unique constraint on `(property_id, phone)`, confirmed live — at-least-once webhook
  delivery makes a same-guest race a real scenario, not a hypothetical.
- **Web-channel joining becomes automatic, conditional on something that doesn't exist yet.** If
  the web widget is ever changed to capture a real phone number, it calls the exact same lookup
  and lands on the exact same row. No bespoke merge step.
- **Retention — decided, not yet built.** A WhatsApp thread with no inbound activity for a
  configurable window (default 10 days) should roll off via an LLM summarization pass, with the
  digest persisted and the raw `MemorySaver` checkpoint cleared. Fully specified in §08, but the
  ticker itself doesn't exist yet — every thread accumulates checkpoint state with no sweep.

---

## §07 — Tool & Capability Parity: The Corrected Inventory

| Layer | What's real and wired up | WhatsApp treatment |
| :--- | :--- | :--- |
| Router | 3-tier dispatch — keyword fast-path, sticky retention, LLM classifier fallback — into `concierge`/`booking`/`dining_spa`/`itinerary`. | Reused as-is — the router doesn't know or care which channel invoked it. |
| Tools (17 real) | 15 live tools bound to subagents; 2 (`get_booking_link`, `get_personal_recommendation`) defined but unbound — dead code. | All 15 live tools reused unchanged. `submit_service_request` gets a real payoff — a WhatsApp-originated Dining request already triggers the shipped staff relay with zero new code. |
| Generative UI (19 payload types) | 15 actually emitted by a tool; 4 (`dining_order_confirmation`, `experience_gallery`, `booking_confirmation`, `timeline_journey`) exist in the renderer's switch but nothing produces them. | Only the 15 real ones need a WhatsApp equivalent — see §08. |
| Memory | `GuestMemoryStore` + `SessionMemoryStore` + LangGraph's own `MemorySaver` — three separate mechanisms, none phone-first. | `thread_id` mapping, property-scoped identity, and phone-based `GuestProfile` linkage are shipped and verified live. Retention/summarization is designed but not built. |
| Entitlements | 5 flags resolved server-side, fail-closed. `itineraryEnabled` and `liveHandoverEnabled` actually gate behavior. | Reused unchanged — a Starter-tier hotel's WhatsApp concierge has exactly the same feature ceiling as its web widget. |
| Streaming | Token-by-token SSE — the core web UX primitive. | No WhatsApp equivalent at all — replies are complete, server-composed messages. The one place WhatsApp is structurally worse than web. |

---

## §08 — WhatsApp-Native Response Design

No official interactive messages exist on this path — no `button`, `interactive`, or `list
message` endpoint anywhere in OpenWA's API. A WhatsApp platform restriction on unofficial
clients, not a gap the adapter can code around.

### Per-UI-type mapping — shipped vs. designed

Only 5 of the 15 real payload types get real image composition today; everything else falls back
to the AI's own plain-text reply. The "numbered text menu" earlier revisions described as the
primary closed-choice mechanism was never actually built as a separate, fabricated message — the
closing message is the AI's natural reply, not a menu generated fresh from payload data.

| Web UI payload | WhatsApp composition | Status |
| :--- | :--- | :--- |
| `rooms`, `room_detail` | One `send-image` per room (photo + `name — price` caption), capped at 5, then the AI's closing reply. | **Shipped** |
| `dining_menu`, `spa_treatment` | One `send-image` per dish/treatment with name + price. | **Shipped** |
| `personal_recommendation` | One `send-image` if the payload has one. | **Shipped** |
| `no_rooms_found`, `weather`, `preferences`, `booking_hold`, `payment_link` | Plain text — the AI's own reply, no media. `preferences` has no chip-picker equivalent, a straight downgrade. | Text-only (as designed) |
| `outlets`, `attractions` | Designed as `send-location` static pins; falls back to plain text today. | **Not built** |
| `itinerary`, `timeline` | Designed as a text sequence with location pins; falls back to plain text today. | **Not built** |
| `human_handover` | Text status ("connecting" vs "logged" per `liveHandoverEnabled`). | Mute shipped · UI not built |

### How images actually move — shipped, matching the verified design

Room/dining images aren't uploaded through any special pipeline — `RoomType.HeroImageURL` and
`AddMedia`-attached assets are plain external URL strings the backend never proxies or re-hosts,
already public, the identical one the web widget puts into an `<img src>` tag.

- The composer takes the URL straight out of the `__UI__` payload — no download/re-upload. **Shipped.**
- `pkg/openwa.SendImage` calls `send-image` directly (`{chatId, url, caption}`) — OpenWA fetches
  it server-side. **Shipped, verified against a real send.**
- Caption capped at OpenWA's real 1024-character limit. **Shipped.**
- Multiple images become a sequence of `send-image` calls, capped at 5 per reply. **Shipped.**
- Base64 fallback for an unreachable URL — **not built.** Every image source in this codebase is
  already public.
- Graceful text-only degradation on a genuine send failure — **not built.** A failed image send
  today aborts the rest of that batch and reports the error.

### Other native primitives, corrected

- **Contact card** (`send-contact`) — name + number only, not a full vCard.
- **Templates** — plain stored text with `{{var}}` substitution, not a Meta-approved rich
  template. No buttons, no approval flow.
- **Presence/typing** — sending works on both engines; *reading* presence is Baileys-only.
- **Location** — static pin only on both engines. No live/continuous sharing.
- **Polls** — engine-dependent. Vote reading is `whatsapp-web.js`-only (Baileys returns `501`),
  and votes match by option *text*, not a stable id. This deployment runs `baileys` for capacity
  reasons — polls can be sent, not meaningfully read back.

### Conversation retention & summarization — mechanics, not yet built

Decided in §06; still unimplemented — every other piece in this section has shipped, this one
hasn't. Designed as a scheduled job (same shape as the capacity monitor's ticker) that: pulls a
thread's checkpointed history, runs a cheap LLM summarization pass extracting durable points,
persists the digest keyed to `(propertyId, guestJID)`, and deletes the raw checkpoint — leaving
the audit-trail `ChatMessageRecord` rows untouched (their retention is a separate, still-open
question). The next message from the same guest seeds context from the digest instead of raw
history.

---

## §09 — Ad Click-Through & Attribution: The Honest Version

The scenario that started this ADR: a guest taps a Click-to-WhatsApp ad and lands in a
conversation with the hotel. **The messaging half of this fully works on Option A; the
attribution half does not.**

**What works:** a Click-to-WhatsApp ad is mechanically just a deep link
(`https://wa.me/<number>?text=<prefilled>`) that opens WhatsApp with a prefilled draft message.
That mechanism doesn't care whether the destination number runs the Cloud API or OpenWA — the
resulting message lands through the same `message.received` webhook as any organic contact.

**What doesn't:** Meta's Cloud API attaches a `referral` object (`source_type`, `source_id`,
`ctwa_clid`) to the first inbound message from a Click-to-WhatsApp ad — a Cloud-API-side
enrichment. OpenWA never parses, maps, or surfaces that field anywhere, confirmed absent across
every one of its 31 docs and its full issue tracker. A hotel on OpenWA cannot distinguish
"came from our Instagram ad" from "messaged us cold" on the same number, full stop.

**What to actually do about it:**
- **Message-content routing** is a real, if partial, substitute — a distinctively-worded prefill
  is recognizable by content, not formal metadata.
- **Per-campaign numbers** get deterministic attribution at real operational/capacity cost.
- **Real campaign-level ROI reporting is Option B/C territory** — the actual trigger for a Cloud
  API migration, not conversation volume or compliance.

### Pre-fill convention — decided, not yet built

The dashboard's WhatsApp integration page should gain a small deep-link generator — a hotel names
a campaign, the tool produces the correct `wa.me` link with a fixed pre-fill pattern. Since the
guest can edit or clear that text, the agent should fall back to a short intent-clarification
question when the first message doesn't match the expected pattern, rather than guessing. Neither
piece is built yet.

---

## §10 — Where MCP Fits

OpenWA exposes a native MCP server (`POST /mcp`, off by default) — 25 read-only tools, or 51 with
writes enabled. The inbound/outbound message loop is not the right place for it — that's a
direct, typed, synchronous call the adapter owns. MCP earns its place one layer up, if the agent
should ever take a WhatsApp action *on its own initiative* (a checkout reminder, a re-engagement
message) rather than a reply to an inbound one. Recommendation: leave it disabled until
agent-initiated outreach is actually scoped.

---

## §11 — Consequences

### Positive
- Inbound webhooks turned out real and fairly complete.
- Guest identity is structurally more durable than the web widget's.
- Zero new agent logic — the same 15 live tools, router, entitlements, RAG carry over untouched.
- `submit_service_request`'s existing staff relay pays off immediately for WhatsApp-originated requests.
- The ad click-through path works as a messaging channel today, at zero extra infrastructure cost.
- WhatsApp became `GuestProfile`'s first genuine population source; future web-side joining falls out for free.
- **The full round trip is live and verified against real traffic** on a real connected number.
- **Two real production bugs surfaced by driving real traffic, not code review** — a dead OpenWA
  session (self-heal shipped) and a channel broadcast mis-classified as a guest (kind-based
  filtering shipped). Evidence that shipping against real traffic early finds real gaps.
- Double-texting and inbound flooding are both handled, shipped and load-bearing on real traffic.
- The AI correctly goes silent on a live handover without ever stranding a Starter/Discover guest.

### Negative
- No ad-attribution metadata on this path, at all.
- Streaming UX has no WhatsApp equivalent.
- Several UI payload types fall back to plain text.
- Poll votes can't be read back on the `baileys` engine this deployment runs.
- No image-send failure degrades gracefully to text yet.
- Conversation retention/summarization is fully designed but not built — threads accumulate indefinitely.
- **The Social Inbox doesn't exist** — a muted guest can only be un-muted by a direct API call today.
- Whether the web widget should ever capture a real phone number is still an open, separate product question.

---

## §12 — Risk Register

| Risk | Probability | Impact | Mitigation |
| :--- | :--- | :--- | :--- |
| WhatsApp protocol change breaks the engine | High | High | Engine is pluggable (`ENGINE_TYPE`); adapter never assumes engine internals. |
| Hotel's number restricted or banned | Medium | Medium–high | Dedicated number, multi-day warm-up, rate limiting, explicit opt-in. |
| Inbound spam/abuse to the connected number | Medium | Low–medium | **Shipped**: fixed per-JID rate cap. Adaptive throttling is future scope. |
| **OpenWA container recreation silently drops session/auth state** | Medium | High (guest number goes dark) | **Real incident, not hypothetical** — see below. |
| **Non-1:1 message kinds miscategorized as guests** | Confirmed occurred once | Low–medium | **Real incident** — see below. |
| No ad-attribution metadata on Option A | Certain | Low–medium | Confirmed absent, not a bug. Message-content routing and per-campaign numbers are partial mitigations. |
| Poll votes unreadable on `baileys` | Certain | Low | Numbered text menus as the primary closed-choice mechanism instead. |
| Session infrastructure cost at fleet scale | Medium | Low–medium | Unchanged capacity monitor, now more load-bearing. |
| Compliance exposure for a regulated hotel group | Real once this ships | High if it applies | Retention/summarization policy (designed, not built) bounds how long raw content persists. |

**OpenWA container data-loss incident, in detail:** recreating the `openwa-api` container to
apply a config change reset its actual session data — the connected number unlinked, and
OpenWA's admin API key itself rotated, which also broke every other property's outbound relay
until the new key was propagated. Shipped mitigation: `ConnectWhatsApp` now calls `GetSession`
before reusing a saved session ID; a 404 clears the stale session fields and falls through to
creating a fresh session, so a hotel gets a real QR to reconnect with instead of a dead end.

**Message-kind misclassification, in detail:** a WhatsApp Channel broadcast arrived with
`isGroup: false`, was accepted as a guest message, created a fake `GuestProfile`/`ChatSession`,
and was forwarded to the agent (which failed sending a reply back to the channel JID). Shipped
mitigation: the receiver now filters on OpenWA's `kind` field, accepting only `"individual"`.

---

## §13 — Rollout

### Phase 1 · shipped — Staff notification relay
OpenWA per hotel, QR-connect, entitlement-gated, category-routed `WhatsAppRecipient` list.

### Phase 2 · shipped — Guest loop closed via web chat
Status changes message the guest back in their own web session — no WhatsApp involved.

### Phase 3 · shipped — Operational hardening
Entitlement-bypass fix, capacity monitoring, Platform Health dashboard, escalation runbook.

### Phase 4 · shipped (core), in progress — Guest-facing WhatsApp concierge

**Shipped:**
- Webhook registration at connect-time + HMAC-verified receiver, with the fixed per-JID rate cap.
- `FindOrCreateGuestByPhone` — `GuestProfile`'s first real writer, verified live.
- Channel-generic `POST /api/messaging/inbound`, non-streaming `hotelAgent` invocation.
- WhatsApp composer for the 5 image-bearing payload types via `SendImage`.
- Per-`(property, phone)` turn serialization, closing a real double-texting race.
- AI-mute on live human handover, correctly excluded for non-live-handover properties.
- Message-`kind` filtering, added after a real incident.
- Self-heal in `ConnectWhatsApp` for a session ID OpenWA no longer recognizes.

**Not built:**
- Conversation retention/summarization ticker.
- Dashboard deep-link generator for the ad pre-fill convention, plus the intent-clarification opener.
- **Social Inbox page** — the single largest remaining piece.
- `SendLocation`/`SendContact`/`SendPoll`/`SetTyping`.
- Pilot with a real Click-to-WhatsApp ad campaign — blocked on the pre-fill generator.
- No auto-escalation for unacknowledged high-priority requests in v1 — decided, not deferred.
- Engine stays fleet-wide `baileys` — per-property `whatsapp-web.js` opt-in is future scope.

### Phase 5 · deferred, optional — Web-side phone capture
Not required for WhatsApp identity to work. The separate, undecided product question of whether
the web widget should ever ask a guest for a real phone number. If built, it joins the same
`GuestProfile` row automatically via the same `FindOrCreateGuestByPhone` lookup.

### Phase 6 · deferred — Official Cloud API migration path
Triggered by real ad-attribution/ROI-reporting need, conversation volume, or a compliance
requirement — whichever comes first. The adapter boundary means this is a new adapter, not a
rewrite of the agent.

---

## §14 — Open Questions

All open questions from earlier revisions are resolved. Nothing remains open as of Revision 7 —
Phase 4's remaining scope (retention ticker, ad pre-fill generator, Social Inbox, remaining
native primitives) is fully specified and buildable, not blocked on any unresolved design
question. Resolved items, for reference:

- ✓ Where does the WhatsApp agent live? Inside `chatbot-demo-admin`, reusing `hotelAgent` non-streaming.
- ✓ Poll or numbered menu for closed-set choices? Numbered menu, primary — polls are engine-dependent.
- ✓ Media round-trips through the same pipeline as the web widget, or simpler? Simpler and direct.
- ✓ Retention policy? 10-day rolloff, LLM-summarize, clear raw checkpoint (designed, not built).
- ✓ Cross-property identity? Always property-scoped, never merged.
- ✓ Engine choice? Fleet-wide `baileys` for v1; per-property opt-in is future scope.
- ✓ Ad creative convention? Standardized deep link + intent-clarification fallback (designed, not built).
- ✓ Auto-escalation for unacknowledged requests? None in v1 — send-once stays as-is.
- ✓ Inbound rate-limiting? Fixed per-JID cap, shipped.
- ✓ Cross-channel guest identity? WhatsApp is `GuestProfile`'s first real writer; web-side joining is free if ever built.

---

## References

**External**
- [rmyndharis/OpenWA](https://github.com/rmyndharis/OpenWA) — repository, README, features, MCP setup
- [docs/06-api-specification.md](https://github.com/rmyndharis/OpenWA/blob/main/docs/06-api-specification.md) — §6.4.2 (messages), §6.4.5 (templates), §6.4.8 (webhook CRUD), §6.6 (webhook envelope, HMAC, delivery semantics)
- [docs/03-system-architecture.md](https://github.com/rmyndharis/OpenWA/blob/main/docs/03-system-architecture.md) — per-session memory guidance, `/api/sessions/stats/overview`
- [docs/16-risk-management.md](https://github.com/rmyndharis/OpenWA/blob/main/docs/16-risk-management.md) — maintainer's own risk register
- [docs/24-mcp-integration.md](https://github.com/rmyndharis/OpenWA/blob/main/docs/24-mcp-integration.md) — MCP tool surface, auth model
- [docs/28-multitenancy.md](https://github.com/rmyndharis/OpenWA/blob/main/docs/28-multitenancy.md) — design proposal, not implemented; no referral/ad-attribution mention
- [Issue #60](https://github.com/rmyndharis/OpenWA/issues/60), [#469](https://github.com/rmyndharis/OpenWA/issues/469), [#127](https://github.com/rmyndharis/OpenWA/issues/127) — community ban-risk reports
- Meta WhatsApp Business Platform documentation — Click-to-WhatsApp ads `referral` webhook object (Cloud-API-only; general platform knowledge, cited for contrast in §09)

**Internal — see `whatsapp_integration_api.md` in this folder** for the full file-level
implementation reference (every controller, model, and route this ADR describes).
