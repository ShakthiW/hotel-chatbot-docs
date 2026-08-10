# Chat Engine & SSE Streaming Gateway API Specification

The **Chat Engine Gateway** handles multi-turn AI Concierge dialogue, Server-Sent Events (SSE) streaming, subagent routing, generative UI tool execution, and session feedback collection.

---

## Endpoints Overview

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/v1/properties/{id}/chat/completions` | Streams real-time Server-Sent Events (SSE) tokens & `__UI__` payloads. |
| `GET` | `/api/v1/properties/{id}/chat/sessions/{sessionId}/history` | Retrieves full conversational message history. |
| `POST` | `/api/v1/properties/{id}/chat/feedback` | Submits guest rating or feedback for a chat session. |

---

## 1. Stream Chat Completions (SSE Gateway)
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/chat/completions` (or `/api/chat` in admin proxy)
* **Headers**: `Content-Type: application/json`, `x-session-id: session-12345`

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

### Event Stream Data Format (`text/event-stream`)
The endpoint streams Server-Sent Events line-by-line:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type":"tool_start","tool":"generate_itinerary"}

data: {"type":"tool_end","tool":"generate_itinerary"}

data: {"type":"ui_payload","payload":{"type":"itinerary","itinerary":{...}}}

data: {"type":"token","text":"Here"}

data: {"type":"token","text":" is"}

data: {"type":"token","text":" a curated 3-day itinerary..."}

data: {"type":"done","threadId":"session-12345","sessionId":"session-12345"}

data: [DONE]
```

---

## 2. Get Chat Session History
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/chat/sessions/{sessionId}/history`

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
