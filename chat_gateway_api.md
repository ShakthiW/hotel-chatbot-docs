# Chat Engine & SSE Streaming Gateway API Specification

The **Chat Engine Gateway** handles multi-turn AI Concierge dialogue, Server-Sent Events (SSE) streaming, subagent routing, generative UI tool execution, and session feedback collection.

> **The real gateway is `POST /api/chat` on `chatbot-demo-admin` (Next.js), not a Go endpoint.**
> A second, independent Gemini-backed chat pipeline used to live at the Go path this document
> originally described (`POST /api/v1/properties/{id}/chat/completions`,
> `chatbot-demo-api/controllers/chat-controller/completions_handler.go`) — it was never called
> by the admin app, had no other caller, and was pure unused surface area, so it was deleted
> outright rather than kept in sync. `chatbot-demo-admin`'s own LangGraph agent
> (`src/app/api/chat/route.ts`) is and always was the actual production chat pipeline; §1 below
> now documents that one directly.

---

## Endpoints Overview

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/chat` (on `chatbot-demo-admin`) | Public | Streams real-time Server-Sent Events (SSE) tokens & `__UI__` payloads from the LangGraph agent. |
| `GET` | `/api/v1/properties/{id}/chat/sessions/{sessionId}/history` | Public | Retrieves full conversational message history. |
| `POST` | `/api/v1/properties/{id}/chat/feedback` | Public | Submits guest rating or feedback for a chat session. |
| `POST` | `/api/v1/properties/{id}/telemetry` | Public | Records one turn's diagnostic trace (see §4 below). |
| `GET` | `/api/v1/properties/{id}/telemetry` | Staff | Retrieves recent traces + aggregate metrics for the dashboard. |

All of section 1–3 are public because they're part of the guest chat flow — guests are never
logged in. `Telemetry` splits the same way as booking: the write is a server-to-server call
from the guest chat backend, the read is a staff dashboard feature.

> **Note on similarly-named routes**: `GET .../chat/sessions/{sessionId}/history` (this
> document, served by `chat-controller`) is a **different** endpoint from
> `GET .../chat-sessions/{sessionKey}/messages` (documented in
> `live_chat_and_handover_websocket_api.md`, served by `session-controller`) — the hyphenation
> and the underlying handler both differ even though the two are easy to conflate by name.
>
> **Note on response envelope**: this document's examples use `{success, data}` — the same
> convention used by `live_chat_and_handover_websocket_api.md` and
> `itinerary_experience_api.md`. This differs from the `{status, message, data}` envelope used
> by `property_management_api.md`, `booking_engine_api.md`, `knowledge_engine_api.md`, and
> `auth_api.md`. Both are real, current conventions — which one you get depends on which
> controller package serves the endpoint, not a documentation typo.

---

## 1. Stream Chat Completions (SSE Gateway)
* **HTTP Method**: `POST`
* **Path**: `/api/chat` — served by `chatbot-demo-admin` itself (`src/app/api/chat/route.ts`),
  not proxied to the Go backend. The Go API is reached only indirectly, via the tool calls this
  route's LangGraph agent makes mid-turn (room lookups, availability checks, knowledge search).
* **Authentication**: Public.
* **Headers**: `Content-Type: application/json`; optional `x-session-id` / `x-guest-id` /
  `x-room-id` (the last one marks the guest as physically resident, injecting in-room context
  and unlocking in-room dining/service requests — see `platform_features.md` §7).

### Request Body
```json
{
  "propertyId": "a8360f9a-445f-405a-9725-232113f382b4",
  "sessionId": "session-a8360f9a-445f-405a-9725-232113f382b4-12345",
  "messages": [
    {
      "role": "user",
      "content": "Can you generate a 3-day romantic itinerary for my stay?"
    }
  ]
}
```

`propertyId` here is the one the route trusts for entitlement-sensitive tools
(`configurable.propertyId`, threaded into every LangGraph tool call) — a tool's own
LLM-supplied `propertyId` argument is optional and unreliable (models frequently omit it), and
was previously falling back to a single hardcoded `DEFAULT_PROPERTY_ID` env var when omitted, a
real cross-tenant data risk fixed by making every tool prefer the request-verified value first.

### Event Stream Data Format (`text/event-stream`)
The endpoint streams Server-Sent Events line-by-line:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type":"thinking","text":"Checking available itinerary slots..."}

data: {"type":"tool_start","tool":"generate_itinerary"}

data: {"type":"tool_end","tool":"generate_itinerary"}

data: {"type":"ui_payload","payload":{"type":"itinerary","itinerary":{...}}}

data: {"type":"token","text":"Here"}

data: {"type":"token","text":" is"}

data: {"type":"token","text":" a curated 3-day itinerary..."}

data: {"type":"subagent_handoff","targetSubagent":"itinerary"}

data: {"type":"done","threadId":"session-12345","sessionId":"session-12345"}

data: [DONE]
```

### Event Type Reference
| Type | Meaning |
| :--- | :--- |
| `thinking` | Model's intermediate reasoning text, shown as an activity indicator. |
| `token` | Streamed narrative text, appended word-by-word. |
| `tool_start` / `tool_end` | A tool call's execution lifecycle (drives loading states). |
| `ui_payload` | Structured card data (`__UI__` payload) for the frontend to render. |
| `subagent_handoff` | The active sub-agent changed (e.g. Concierge → Booking) mid-conversation. |
| `done` | Stream complete; carries the final `threadId`/`sessionId`. |
| `error` | A terminal error occurred; carries a `message` field. |
| `[DONE]` | Literal connection-close sentinel (not JSON), always the final line. |

On a hard failure the stream instead emits a single `{"type":"error","message":"..."}` frame
in place of further `token`/`done` frames.

### `ui_payload` Type Catalog

Every LangGraph tool that has something structured to show embeds a
`__UI__{"type": "...", ...}__UI__` marker in its return string; `route.ts` regex-extracts it
and emits it as its own `ui_payload` SSE frame, separate from the narrated `token` text. This
is the complete list of `type` values a tool can currently emit — verified against
`chatbot-demo-admin/src/lib/agent/tools/index.ts`, not aspirational:

| `type` | Emitted By | Shape Notes |
| :--- | :--- | :--- |
| `rooms` | `check_room_availability` | `rooms[]`, each with `recommendationReason` (why it matched what was asked) and `badges[]` (channel badges, plus a real "Only N left" badge when true remaining inventory is 2 or fewer). Also carries top-level `appliedFilters` (`viewType`/`adults`/`children`) so the UI can show what was actually filtered on. Rooms are pre-sorted by `sensoryMatchScore` descending. |
| `no_rooms_found` | `check_room_availability` | Hard filters (capacity, a named view) excluded every room — carries the same `appliedFilters` plus `maxAdultsInCatalog`/`maxChildrenInCatalog`/`availableViewTypes` so the UI can suggest something concrete instead of a dead end. This is a distinct outcome from an error; the tool never silently falls back to an unfiltered room list. |
| `outlets` | `get_outlet_details` | Dining/spa outlet cards. |
| `dining_menu` | `get_dining_menu` | Menu items with dietary tags, chef-recommendation flag. |
| `spa_treatment` | (via `get_dining_menu`'s spa branch) | Treatment cards with duration, focus area. |
| `attractions` | `get_destination_attractions` | Distance/transport/best-time cards with a Google Maps deep link. |
| `weather` | (weather tool) | Current conditions + AI stay-advice narrative. |
| `personal_recommendation` | `get_personal_recommendation` | Opinionated single-item pick with `matchReasons[]`. |
| `preferences` | `save_guest_preferences` | Renders the (client-hardcoded, not server-sent) preference chip picker; carries `initialSelectedIds[]` only. **Currently only wired to the `dining_spa` subagent** — a guest asking the general concierge to save preferences will be told it isn't possible, which is a known open gap, not by design. |
| `booking_hold` | `create_room_hold` | Active 15-minute hold confirmation. |
| `payment_link` | `generate_payment_link` | Checkout URL, amount, expiry. |
| `itinerary` | `generate_itinerary` | Multi-day day-by-day plan. |
| `experience_timeline` | `get_experience_timeline` | Hour-by-hour single-day slots. |
| `human_handover` | `request_human_handover` | `status: "logged"` (no live-handover entitlement) or `"connecting"` (live handover), with a guest-facing `message`. |

**Rendering parity is not automatic.** The admin dashboard's Live Preview
(`WidgetRenderer.tsx`) and the real in-room QR guest chat (`GuestChatWidget.tsx`, which shares
the same renderer) handle all of the above. The embeddable script real hotel websites load
(`chatbot-demo-admin/public/widget.js`) is a separate, hand-written vanilla-JS renderer that
has to be kept in sync by hand — see `widget_embed_architecture.md` for its current coverage
and the drift risk that comes with maintaining two renderers for one payload contract.

---

## 2. Get Chat Session History
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/chat/sessions/{sessionId}/history`
* **Authentication**: Public.

### Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "session_id": "session-12345",
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "messages": [
      {
        "id": "msg-001",
        "role": "user",
        "content": "Can you generate a 3-day romantic itinerary for my stay?",
        "created_at": "2026-08-06T15:00:00Z"
      },
      {
        "id": "msg-002",
        "role": "assistant",
        "content": "Here is a beautifully curated 3-day romantic itinerary...",
        "created_at": "2026-08-06T15:00:05Z"
      }
    ]
  }
}
```

---

## 3. Submit Chat Feedback
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/chat/feedback`
* **Authentication**: Public.

### Request Body
```json
{
  "session_id": "session-12345",
  "rating": 5,
  "comment": "Exquisite itinerary recommendation!"
}
```

### Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "message": "Feedback submitted successfully"
  }
}
```

---

## 4. Telemetry (Per-Turn Diagnostic Trace)

Every guest chat turn is recorded as a telemetry entry — user query, guardrail outcome,
subagent used, tools executed, latency, and token counts — persisted server-side so the
dashboard's Telemetry page survives serverless cold starts (it previously lived only in an
in-memory, per-instance log).

### 4.1 Record Telemetry Entry
Written server-to-server by the guest chat backend after every turn; not intended to be
called directly by a client.

* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/telemetry`
* **Authentication**: Public (guest-flow, same tier as `tokens/consume`).

#### Request Body
```json
{
  "thread_id": "session-a8360f9a-1724678400000",
  "user_query": "Do you have an ocean view suite available?",
  "guardrail_status": "passed",
  "guardrail_reason": "",
  "subagent_used": "booking",
  "tools_executed": ["check_room_availability"],
  "execution_time_ms": 850,
  "assistant_response": "Yes! The Ocean View Suite is available...",
  "prompt_tokens": 620,
  "completion_tokens": 140,
  "generative_ui_count": 1,
  "generative_ui_tokens": 1000,
  "total_tokens": 1760
}
```

### 4.2 Get Telemetry
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/telemetry`
* **Authentication**: Required — staff JWT scoped to this property.

#### Response (`200 OK`)
```json
{
  "success": true,
  "message": "Telemetry retrieved successfully",
  "data": {
    "logs": [
      {
        "id": "e4687aa5-96ce-4058-acc3-3824bfb2123c",
        "timestamp": "2026-08-29T07:01:45Z",
        "threadId": "session-a8360f9a-1724678400000",
        "propertyId": "a8360f9a-445f-405a-9725-232113f382b4",
        "userQuery": "Do you have an ocean view suite available?",
        "guardrailStatus": "passed",
        "subagentUsed": "booking",
        "toolsExecuted": ["check_room_availability"],
        "executionTimeMs": 850,
        "assistantResponse": "Yes! The Ocean View Suite is available...",
        "totalTokens": 1760
      }
    ],
    "metrics": {
      "total": 1,
      "guardrailPassRate": 100,
      "avgLatencyMs": 850,
      "subagentCounts": { "concierge": 0, "booking": 1, "dining_spa": 0, "itinerary": 0 },
      "totalTokens": 1760,
      "avgTokensPerTurn": 1760,
      "totalGenerativeUiTokens": 1000
    }
  }
}
```
Note the `logs` entries use camelCase field names — this endpoint intentionally reshapes the
underlying (snake_case) database columns to match the dashboard's existing `TelemetryLog`
type, so no frontend changes were needed when this moved from an in-memory store to
persistent storage.
