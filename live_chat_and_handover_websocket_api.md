# Live Chat, Human Handover & Real-Time WebSocket API Specification

The **Live Chat & Human Handover Subsystem** provides sub-50ms bidirectional communication between hotel guests (via the Web Widget) and front-desk concierge operators (via the Live Concierge Desk in the Admin Dashboard). 

It orchestrates seamless transitions between the **Autonomous AI Concierge** and **Human Staff**, with zero-downtime reconnects, multi-session operator triage, staff whisper notes, and AI copilot reply suggestions.

---

## 🏗️ Architecture & Component Topology

```mermaid
flowchart TD
    subgraph GuestLayer [Guest Client Layer]
        Widget[Web Chat Widget / Simulator]
    end

    subgraph AdminLayer [Staff & Concierge Layer]
        Dashboard[Unified Conversations Dashboard]
        GlobalListener[Global Session Multi-Queue Listener]
    end

    subgraph BackendLayer [Go WebSocket Hub (chatbot-demo-api)]
        Hub[Gorilla WebSocket Hub Engine]
        Rooms[In-Memory Session Rooms map[sessionKey]map[*Client]bool]
        GlobalRoom[Global Operator Channel]
    end

    subgraph DataLayer [Storage & Persistence]
        Postgres[(PostgreSQL: chat_sessions & chat_messages)]
        LangGraph[LangGraph Multi-Agent Orchestrator]
    end

    Widget <-->|WS: /api/v1/ws/sessions/{key}| Hub
    Dashboard <-->|WS: /api/v1/ws/sessions/{key}| Hub
    GlobalListener <-->|WS: /api/v1/ws/sessions/global?role=operator| Hub

    Hub --- Rooms
    Hub --- GlobalRoom

    Hub -->|Async Non-Blocking Writes| Postgres
    Widget -.->|POST /api/chat when AI Active| LangGraph
    LangGraph -.->|POST /chat-sessions/{key}/messages| Postgres
```

---

## 📡 1. WebSocket Protocol Specification

### Endpoints
| Protocol | Path | Role | Description |
| :--- | :--- | :--- | :--- |
| `WS` | `/api/v1/ws/sessions/{sessionKey}` | `guest` / `operator` | Peer-to-peer room for active guest and operator communication. |
| `WS` | `/api/v1/ws/sessions/global?role=operator` | `operator` | Global stream delivering state changes, new messages, and handover alerts across **all active sessions** in the property. |

### Query Parameters
* `role`: `"guest"` | `"operator"` (Default: `"guest"`)
* `name`: Display name of the participant (e.g., `"Shakthi (Front Desk)"` or `"Guest"`).
* `propertyId`: UUID of the active property for multi-tenant boundary isolation.
* `token`: **Required when `role=operator`**, ignored for `role=guest`. A staff JWT
  (the same token used for REST calls), passed as a query parameter because browsers cannot
  set custom headers on a WebSocket upgrade request. The connection is rejected (`401`) if the
  token is missing/invalid, and rejected (`403`) if the token's `property_id` claim doesn't
  match `propertyId` (unless the token's role is `super_admin`). Guest connections have no
  equivalent check — the `sessionKey` itself is the only thing gating access to that room, so
  treat session keys as capability tokens, not public identifiers.

> **Authentication summary for this document**: guest WebSocket connections and the guest-only
> REST calls (`GET`/`POST .../messages`, `POST .../request-handover`) are public — guests are
> never logged in. Operator WebSocket connections require the `token` parameter above.
> Everything else in the REST section (list sessions, takeover, handback, suggested-replies,
> delete) requires a staff JWT scoped to the property.
>
> **Note on similarly-named routes**: the `chat-sessions` endpoints here (served by
> `session-controller`) are distinct from `GET .../chat/sessions/{sessionId}/history` (served
> by `chat-controller`, documented in `chat_gateway_api.md`) — same-sounding names, different
> handlers and different path segments (`chat-sessions` vs `chat/sessions`).
>
> **Note on response envelope**: REST responses in this document use `{success, data}` /
> `{success: false, error}`, matching `chat_gateway_api.md` and `itinerary_experience_api.md`
> — see the envelope note there for why this differs from other docs in this set.

---

## 💬 2. JSON Frame Format & Event Types

Every payload exchanged over the WebSocket connection adheres to the standard `WSMessage` schema:

```json
{
  "type": "chat_message",
  "session_key": "session-a8360f9a-1724678400000",
  "sender": "operator",
  "sender_name": "Dilshan (Concierge)",
  "content": "Good afternoon! I have arranged complimentary late checkout at 2:00 PM for you.",
  "is_whisper": false,
  "is_typing": false,
  "state": "human_active",
  "payload_type": "text",
  "payload_json": "",
  "timestamp": "2026-08-26T14:30:00Z"
}
```

### Event Type Matrix

| Type | Direction | Description |
| :--- | :--- | :--- |
| `chat_message` | Bidirectional | Standard user/operator message turn. Delivered to room peers and persisted to PostgreSQL. |
| `typing` | Bidirectional | Ephemeral typing indicator. Broadcast with zero DB overhead; auto-cleared by peer timeout. |
| `presence` | Server $\rightarrow$ Client | Emitted when a participant connects or disconnects from the session. |
| `state_change` | Bidirectional | Triggers mode change (`ai_active`, `handover_requested`, `human_active`, `resolved`). |
| `session_deleted` | Server $\rightarrow$ Client | Broadcast when an operator permanently purges a conversation. Removes card from all active dashboards. |
| `suggested_replies`| Server $\rightarrow$ Operator | Transmits AI Copilot suggested response drafts tailored to the conversation context. |

---

## 🔄 3. Human Handover Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> ai_active: Guest opens chat
    ai_active --> handover_requested: Guest types "talk to human" OR AI triggers handover tool
    handover_requested --> human_active: Operator clicks "Take Over Live"
    human_active --> ai_active: Operator clicks "Hand Back to AI"
    human_active --> resolved: Operator marks issue resolved
    resolved --> ai_active: Guest resumes conversation
```

### Mode Descriptions
1. **`ai_active`**: The LangGraph multi-agent hierarchy (Concierge, Reservations, Dining/Spa, Itinerary) handles guest queries autonomously via streaming SSE.
2. **`handover_requested`**: The guest requested human assistance or the AI guardrails escalated the conversation. The dashboard sounds a luxury audio chime and flags the session in the **Needs Action** triage tab.
3. **`human_active`**: A staff operator has accepted the session. The AI agent pauses inference on that thread, and all messages route directly between guest and human operator via WebSocket.
4. **`resolved`**: Live assistance concluded. The conversation is archived and memory extraction updates the guest profile.

> **WhatsApp sessions (`channel: "whatsapp"`) follow this same state machine, with two real
> differences** — see `whatsapp_integration_api.md` §5 for the full picture:
> - **Where the AI-pause is actually enforced.** For `channel: "web_chat"`, "the AI agent pauses
>   inference" is a property of how the web widget's own route is wired. For WhatsApp, the pause
>   is a hard gate in `chatbot-demo-api`'s inbound webhook receiver — it checks `session_mode`
>   before ever forwarding a message to the agent, so a guest's follow-up message while
>   `handover_requested`/`human_active` is persisted but genuinely never reaches the LLM.
> - **Not every handover request flips the mode.** `request_human_handover` only triggers
>   `handover_requested` when the property has `liveHandoverEnabled` (the tool's `"connecting"`
>   outcome). A Starter/Discover property gets the tool's `"logged"` outcome instead, and the
>   session mode is deliberately left untouched — that property has no `handback` access either
>   (both are gated behind the same entitlement), so flipping the mode there would strand the
>   guest with no way for anyone to ever resume the AI.
> - **No WebSocket relay exists for WhatsApp yet.** "All messages route directly... via
>   WebSocket" in point 3 above is web-widget-specific. A `human_active` WhatsApp session has no
>   staff-facing UI to type a reply into — the Social Inbox that would provide one isn't built.
>   Today, the only way to move a WhatsApp session back out of `human_active` (or
>   `handover_requested`) is a direct, staff-authenticated call to §5.6's `handback` endpoint.

---

## 🔒 4. Staff Internal Whispers (Private Notes)

Operators can leave internal notes during live sessions (e.g. *"Guest requested room upgrade on anniversary"*):
* In `WSMessage`, set `"is_whisper": true` with `"sender": "operator"`.
* **Security Filter**: The Go Hub automatically skips sending whisper payloads to any client connected with `role != "operator"`.
* Whispers are highlighted with an amber border and lock icon in the staff console and are completely invisible to the guest.

---

## 🛠️ 5. REST Control Endpoints

### 1. List Active Chat Sessions
* **Endpoint**: `GET /api/v1/properties/{id}/chat-sessions`
* **Authentication**: Required — staff JWT scoped to this property. This returns every guest's
  session metadata for the property, so it is intentionally staff-only (see §2 below for the
  public, single-session alternative guest widgets use instead).
* **Query Params**: `session_mode` (`All`, `handover_requested`, `human_active`, `ai_active`), `channel` (`web_chat`, `whatsapp`)
* **Response**:
```json
{
  "success": true,
  "data": [
    {
      "id": "e391b10a-3c5e-49b8-a764-cf36f251c142",
      "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
      "session_key": "session-a8360f9a-1724678400000",
      "guest_name": "Elena Rostova",
      "room_number": "Villa 402",
      "channel": "web_chat",
      "session_mode": "handover_requested",
      "assigned_operator": "",
      "last_message_at": "2026-08-26T14:28:10Z",
      "created_at": "2026-08-26T14:15:00Z"
    }
  ]
}
```

---

### 2. Get One Chat Session (Guest-Safe Status Lookup)
* **Endpoint**: `GET /api/v1/properties/{id}/chat-sessions/{sessionKey}`
* **Authentication**: Public. This exists specifically so a guest widget can learn its own
  session's live-handover status (`session_mode`, `assigned_operator`) without being able to
  see every other guest's session, which is what calling §1's list endpoint would require.
* **Response**:
```json
{
  "success": true,
  "data": {
    "id": "e391b10a-3c5e-49b8-a764-cf36f251c142",
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "session_key": "session-a8360f9a-1724678400000",
    "guest_name": "Elena Rostova",
    "session_mode": "human_active",
    "assigned_operator": "Dilshan (Duty Manager)",
    "handover_reason": "Guest asked for manager regarding villa check-in time",
    "last_message_at": "2026-08-26T14:28:10Z"
  }
}
```

---

### 3. Get / Save Session Messages
* **Endpoints**: `GET /api/v1/properties/{id}/chat-sessions/{sessionKey}/messages`,
  `POST /api/v1/properties/{id}/chat-sessions/{sessionKey}/messages`
* **Authentication**: Public — both are called by the guest chat backend and, for `GET`, by
  guest widgets directly. Internal staff whispers (`is_whisper: true` records) are only
  included in the `GET` response when the caller presents a valid staff JWT scoped to this
  property (query param `?role=operator` alone is **not** sufficient — it must be paired with
  a real `Authorization: Bearer <token>` header, precisely to prevent a guest from simply
  appending `?role=operator` to read staff notes).
* **`GET` Response**:
```json
{
  "success": true,
  "data": [
    { "id": "m1", "sender": "guest", "content": "Do you have a room with a sea view?", "is_whisper": false, "created_at": "2026-08-26T14:15:03Z" },
    { "id": "m2", "sender": "ai", "content": "Yes! The Ocean View Suite...", "is_whisper": false, "created_at": "2026-08-26T14:15:06Z" }
  ]
}
```
* **`POST` Request Body**:
```json
{
  "sender": "guest",
  "content": "Do you have a room with a sea view?",
  "channel": "web_chat"
}
```

---

### 4. Request Human Handover (Guest/AI Trigger)
* **Endpoint**: `POST /api/v1/properties/{id}/chat-sessions/{sessionKey}/request-handover`
* **Authentication**: Public — triggered by the guest chat backend, not a logged-in user.
* **Request Body**:
```json
{
  "reason": "Guest asked for manager regarding villa check-in time",
  "guest_name": "Elena Rostova",
  "room_number": "Villa 402"
}
```
* **Effects**:
  - Updates `session_mode` in database to `handover_requested`.
  - Broadcasts `state_change` across the session room and the `global` operator stream.
  - Generates an audible chime on all open concierge dashboards.

---

### 5. Takeover Session (Operator Action)
* **Endpoint**: `POST /api/v1/properties/{id}/chat-sessions/{sessionKey}/takeover`
* **Authentication**: Required — staff JWT scoped to this property.
* **Request Body**:
```json
{
  "operator_name": "Dilshan (Duty Manager)"
}
```
* **Effects**:
  - Sets `session_mode` to `human_active` and `assigned_operator` to the staff member's name.
  - Broadcasts `state_change` to the guest widget, changing widget banner to *"Connected with Dilshan (Duty Manager)"*.

---

### 6. Hand Back to AI (Operator Action)
* **Endpoint**: `POST /api/v1/properties/{id}/chat-sessions/{sessionKey}/handback`
* **Authentication**: Required — staff JWT scoped to this property.
* **Effects**:
  - Sets `session_mode` to `ai_active`.
  - Sends system transition message to guest widget: *"Operator has concluded live assistance. AI Concierge is now active."*

---

### 7. Fetch Suggested AI Copilot Replies
* **Endpoint**: `POST /api/v1/properties/{id}/chat-sessions/{sessionKey}/suggest-replies`
  (note: `POST`, and `suggest-replies` — not `GET`/`suggested-replies`)
* **Authentication**: Required — staff JWT scoped to this property.
* **Response**:
```json
{
  "success": true,
  "data": [
    "I would be delighted to arrange complimentary late checkout at 2:00 PM for you.",
    "Let me check with our housekeeping team right away and confirm your request.",
    "Our private dining team is preparing your table now. Is there anything else you require?"
  ]
}
```

---

### 8. Delete Chat Session (Permanent Purge)
* **Endpoint**: `DELETE /api/v1/properties/{id}/chat-sessions/{sessionKey}`
* **Authentication**: Required — staff JWT scoped to this property.
* **Effects**:
  - Deletes all associated records in `chat_messages` and `chat_sessions`.
  - Broadcasts `session_deleted` event so all open dashboards remove the conversation from view instantly.
