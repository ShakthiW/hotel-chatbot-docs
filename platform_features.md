# AI Hospitality Experience Platform — Complete Feature Compendium

> **The platform philosophy in one sentence:**  
> Every conversation should feel like speaking with the hotel's best concierge — not reading a brochure.

This document is a full audit of every feature across the **`chatbot-demo-admin`** (Next.js 15 frontend + LangGraph agent) and the **`chatbot-demo-api`** (Go REST backend), enriched with the storytelling dimension that makes the product category-defining.

---

## Table of Contents

1. [Multi-Agent AI Architecture](#1-multi-agent-ai-architecture)
2. [Dual-Track Guardrail System](#2-dual-track-guardrail-system)
3. [Storytelling & Sensory Experience Engine](#3-storytelling--sensory-experience-engine)
4. [Knowledge Engine (RAG Pipeline)](#4-knowledge-engine-rag-pipeline)
5. [Generative UI — Conversational Rich Cards](#5-generative-ui--conversational-rich-cards)
6. [Conversational Booking Engine](#6-conversational-booking-engine)
7. [Dining, Wellness & Spa Intelligence](#7-dining-wellness--spa-intelligence)
8. [Destination & Attraction Layer (Sri Lanka Experience)](#8-destination--attraction-layer-sri-lanka-experience)
9. [AI Itinerary Generator](#9-ai-itinerary-generator)
10. [Hour-by-Hour Resort Experience Timeline](#10-hour-by-hour-resort-experience-timeline)
11. [Memory Engine — Dual-Tier Guest Intelligence](#11-memory-engine--dual-tier-guest-intelligence)
12. [Bot Persona Configuration Studio](#12-bot-persona-configuration-studio)
13. [Property Management System (PMS Backend)](#13-property-management-system-pms-backend)
14. [Seasonal Intelligence Layer](#14-seasonal-intelligence-layer)
15. [Heritage Story Builder](#15-heritage-story-builder)
16. [Chat Gateway & SSE Streaming](#16-chat-gateway--sse-streaming)
17. [Hotel Dashboard & Admin Workspace](#17-hotel-dashboard--admin-workspace)
18. [Vercel Blob Cloud Storage & Media Streaming Architecture](#18-vercel-blob-cloud-storage--media-streaming-architecture)
19. [Multi-Tenant Token Budgeting & Automated Monthly Reset](#19-multi-tenant-token-budgeting--automated-monthly-reset)
20. [Digital Concierge Studio 2.0 & Live Preview](#20-digital-concierge-studio-20--live-preview)
21. [How All Features Complement One Another](#21-how-all-features-complement-one-another)

---

## 1. Multi-Agent AI Architecture

### What It Is
Instead of a single monolithic AI model, the platform uses a **LangGraph-powered multi-agent StateGraph** compiled with a persistent thread checkpointer. Four specialised sub-agents work together under a Supervisor Router.

### The Four Sub-Agents

| Sub-Agent | Speciality | Handoff Triggers |
|---|---|---|
| **Concierge** | Heritage stories, local attractions, Sri Lanka knowledge, seasonal guides, FAQs | → Booking (room inquiry) → Dining (menu/spa) |
| **Booking** | Room categories, rates, price breakdowns, holds, payment links | → Concierge (attractions/history) → Dining (menu) |
| **Dining & Spa** | Restaurant outlets, cuisine types, dietary options, spa packages | → Booking (room reservation) → Concierge (general) |
| **Itinerary** | Multi-day trip plans, 24-hour sensory timelines, experience curation | → Booking (ready to reserve) → Dining (menu questions) |

### How the Supervisor Router Works

Every incoming guest message is first evaluated by a two-layer classifier:
1. **Fast keyword pre-check** (sub-millisecond): Deterministic pattern matching on words like "room", "spa", "itinerary", "book".
2. **LLM structured output fallback**: `gemini-2.5-flash` with `withStructuredOutput` to classify intent when keywords are ambiguous.

The active sub-agent state persists across turns — if a guest is mid-booking flow, subsequent replies continue in the Booking agent even without explicit booking keywords.

### Why This Matters for Storytelling
Each sub-agent carries its own **system prompt personality** and tool palette. The Concierge agent tells stories about the hotel's heritage; the Booking agent narrates why dates are priced higher using seasonal context; the Itinerary agent weaves hotel experiences, local culture, and wildlife into a day-by-day narrative. The guest experiences seamless handoffs — not jarring topic changes.

---

## 2. Dual-Track Guardrail System

### What It Is
A two-stage input safety layer that fires before any sub-agent sees the message.

### Track 1 — Fast Deterministic Pre-Scanner (< 1ms)
A regex/pattern-based scanner checks for:
- Prompt injection patterns
- Jailbreak attempts
- Hard-coded threat signatures
- Code generation requests (Python, SQL, scripts)

If a hard threat is detected, it returns immediately without any LLM call.

### Track 2 — Sliding Context Window LLM Evaluator
A 4-message conversation window is passed to `gemini-2.5-flash` using `withStructuredOutput`. The model classifies the input into one of four dialogue types:

| Classification | Action |
|---|---|
| `new_hospitality_query` | Allow — pass to sub-agent |
| `guest_detail_response` | Allow — the guest is replying to an agent question (e.g. providing their email) |
| `off_topic_request` | Reject — redirect warmly |
| `malicious_injection` | Reject — log and decline |

The guardrail is **context-aware** — it understands that "Jane Doe, jane@email.com" is a valid response to a booking question, not a random off-topic message.

### The Refusal Node
When the guardrail rejects a query, a dedicated **Refusal Node** generates a warm, brand-voice-appropriate response that:
- Acknowledges the guest politely
- Declines to help with the out-of-scope topic
- Invites them back to explore the hotel, dining, spa, or local experiences

The refusal itself carries the hotel's brand tone — it is never cold or robotic.

---

## 3. Storytelling & Sensory Experience Engine

### The Core Differentiator
This is what separates the platform from every other hotel chatbot. Rather than answering *"What is the Ocean View Suite?"* with room specifications, the AI responds with:

> *"Around sunrise you'll usually hear the waves before you see them. Guests often mention opening the balcony doors while enjoying a coffee and watching fishing boats head out across calm morning waters."*

### How It's Implemented

**In the backend:**
Every `RoomType` entity in PostgreSQL carries a `sensory_experience` text field — a curated first-person sensory narrative written specifically for that room. When the RAG system embeds this room's knowledge, the sensory experience is included in the parent context block.

**In the AI agent:**
The Concierge sub-agent's system prompt instructs the AI to:
- Combine factual information with carefully crafted storytelling
- Describe what guests *hear*, *feel*, *smell*, and *see*
- Use `search_hotel_knowledge` to retrieve sensory narratives from Qdrant before composing responses

**In the bot config:**
Hotel staff choose a `response_detail` level:
- `rich` — evocative storytelling details that build excitement
- `balanced` — informative with some atmosphere
- `concise` — direct and to the point

### What Gets Storytelling Treatment

| Entity | Storytelling Dimension |
|---|---|
| Room Types | Sensory experience field: sounds, views, morning light, balcony feel |
| Heritage Stories | Hotel history, local legends, architecture, chef stories |
| Seasonal Guides | Whale watching, monsoon moods, turtle nesting, festival atmosphere |
| Experience Timeline | Hour-by-hour sensory slots with crowd levels and ambient descriptions |
| Destination Attractions | Cultural context, best photography moments, hidden local knowledge |

---

## 4. Knowledge Engine (RAG Pipeline)

### What It Is
A multi-tenant Retrieval-Augmented Generation (RAG) pipeline that converts unstructured hotel documents and structured PMS entities into searchable vector knowledge.

### Ingestion Pipeline

```
PDF / Markdown / Text → Parser → Hierarchical Chunker → Gemini Embedder → Qdrant Cloud
PMS Entities (Rooms, Outlets, Stories, Seasons, Events) → Auto-Sync → Gemini Embedder → Qdrant Cloud
```

### Chunking Strategy

| Level | Size | Purpose |
|---|---|---|
| Child Chunk | ~300 characters, 50-char overlap | High-precision semantic search unit |
| Parent Content | ~1500 characters | Rich context block returned to LLM to avoid fragmented answers |

### Embedding Model: `gemini-embedding-2`
- **3072-dimensional** vectors (highest available precision)
- **Cosine distance** metric in Qdrant Cloud
- Task-prefixed prompting for improved retrieval accuracy:
  - Ingestion: `title: {source_title} | text: {chunk_content}`
  - Query: `task: question answering | query: {search_query}`

### Multi-Tenant Isolation
Every vector point is tagged with `property_id`. All Qdrant searches include a mandatory `property_id` filter — no cross-property knowledge leakage is possible.

### Real-Time Auto-Sync
When hotel staff update or delete an **attraction, seasonal guide, or event** in the admin
dashboard, the backend **automatically triggers a knowledge base re-sync** for that entity:
- Updated entities → re-embed and upsert into Qdrant
- Deleted entities → purge all associated vectors from Qdrant and PostgreSQL

Rooms and stories are *not* included in this — there is no auto-sync on room or story
mutations today (verified against `controllers/property-controller/`: only
`attraction_handler.go`, `season_handler.go`, and `event_handler.go` call the sync path).
Media has a separate, related mechanism instead: `POST .../media/sync` runs Gemini Vision
analysis over media assets and indexes the resulting descriptions.

### How Storytelling Connects
The RAG engine stores sensory experience narratives, heritage stories, and seasonal mood descriptions alongside factual data. When a guest asks an emotional question ("What will I feel waking up there?"), the vector search retrieves the sensory narrative — not just the room dimensions.

---

## 5. Generative UI — Conversational Rich Cards

### What It Is
The AI doesn't just return text. It returns structured `__UI__` payloads embedded in tool responses that the frontend parses and renders as rich interactive components alongside the narrative text.

### The UI Card Library

| Card Component | Triggered By | What It Shows |
|---|---|---|
| **RoomCarouselUI** | `check_room_availability` tool | Adaptive room cards with sensory descriptions, dynamic preferred channel CTA (`Book Direct`, `Book on Booking.com`, `Book on Agoda`), multi-channel popover dropdown, and outbound click tracking. |
| **RoomDetailUI** | Room card expansion | High-resolution modal with complete sensory description, dimensions, view specs, verified badges, and dynamic booking actions. |
| **RoomInquiryModal** | Fallback / Zero-link state | Direct concierge room inquiry modal pre-populating room code, requested stay dates, guest count, and special requests. |
| **DiningOutletUI** | `get_outlet_details` tool | Restaurant/spa cards with cuisine, dress code, dietary tags, location |
| **AttractionCardUI** | `get_destination_attractions` tool | Attraction cards with distance, transport, best time to visit, Google Maps link |
| **ItineraryCardUI** | `generate_itinerary` tool | Multi-day timeline with activity cards per day, source badges, price info |
| **ExperienceTimelineUI** | `get_experience_timeline` tool | Hour-by-hour resort timeline with crowd levels and sensory highlights |
| **BookingHoldCardUI** | `create_room_hold` tool | Active hold confirmation with room, hold ID, 15-minute countdown (Integrated Mode) |
| **PaymentCheckoutCardUI** | `generate_payment_link` tool | Direct checkout link, total amount, QR code, expiry |
| **TimelineJourneyCardUI** | Custom journey flows | Visual journey progression across the guest's stay narrative |
| **ThinkingAccordion** | Model thinking mode | Collapsible reasoning trace for transparency |

### How It Works Technically
Tool responses include a `__UI__${JSON.stringify(payload)}__UI__` sentinel. The frontend parses this pattern from SSE token streams and renders the appropriate React component inline within the conversation — while the agent's natural language text wraps around it.

### Why This Is Powerful
A guest asks: *"Show me your available rooms."*
The agent responds with an evocative two-sentence sensory description AND renders an interactive carousel of room cards with full details. The story and the data arrive together.

---

## 6. Omnichannel Booking Engine & Outbound Intent Flow

### What It Is
In **holaa V1**, the platform operates in **External Booking Engine Mode** (`booking_mode: "external"`), providing a production-safe, high-converting external booking workflow that directs guests to the hotel's configured booking destinations (Direct Hotel Website, Booking.com, Agoda, Expedia, or Custom Portals) with complete outbound intent tracking.

### The External Booking Flow
```
1. check_room_availability     → 4-Tier Precedence Resolver (Override > Property Default > Auto OTA > Inquiry)
        ↓
2. RoomCarouselUI Rendering    → Dynamic Primary CTA ("Book Direct") + Multi-Channel Popover
        ↓
3. Guest Clicks Booking CTA    → Asynchronous Outbound Intent Logging (POST /booking/track-click)
        ↓
4. Outbound Redirect           → Safe new-tab navigation with pre-filled check-in, check-out, and guest parameters
```

### Key Booking & Distribution Features

**Omnichannel Destination Management:** Hoteliers configure property-level destinations in `/dashboard/integrations` (Direct Website with 0% OTA commission, Booking.com, Agoda, Expedia, Other) and set their preferred primary channel.

**Room-Level Override Hierarchy:** Specific room categories can define custom checkout links in `/dashboard/rooms` and `/dashboard/experience` that override property-wide defaults.

**Universal Legacy Fallback:** If no custom URLs are configured, holaa automatically constructs deep-links to the hotel's Booking.com property page using its slug and affiliate AID. If no slug exists, it seamlessly opens the **Room Inquiry Modal**.

**Outbound Click Tracking & Intent Analytics:** Tracks guest clicks across channels, computes room CTR %, and aggregates outbound booking traffic in `/dashboard/analytics` without making false claims of live PMS room locks.

**Safe V1 Hospitality AI Language:** Prompts enforce that the AI concierge guides guests to verified booking portals and accurately calculates multi-night totals without claiming unverified inventory locks.

**Integrated Mode & 15-Minute Hold Simulator (Dev/Future):** Retains full support for 15-minute temporary room locks (`create_room_hold`), price breakdown calculations (`get_price_breakdown`), and checkout simulators (`/buy`) for future live PMS bridges.

---

## 7. Dining, Wellness & Spa Intelligence

### What It Is
Every dining outlet, spa, bar, pool, and wellness facility is stored as a structured entity in PostgreSQL and surfaced conversationally.

### Outlet Data Model

| Field | Examples |
|---|---|
| `outlet_type` | Restaurant, Bar, Spa, Pool, Gym, Tea Lounge |
| `cuisine_type` | Sri Lankan, Seafood, International, Tea Ceremony |
| `dietary_tags` | Halal, Vegan, Gluten-Free |
| `dress_code` | Smart Casual, Resort Casual, Formal |
| `requires_reservation` | Yes / No |
| `operating_hours` | JSON schedule per day |
| `location_on_property` | Poolside, Oceanfront, Garden Terrace |

### The Dining & Spa Sub-Agent
On every activation, the Dining sub-agent **immediately calls `get_outlet_details`** (this is a hard instruction in the prompt template) to fetch live outlet data before composing any response. This ensures information is always current — not hallucinated.

The resulting `DiningOutletUI` card renders all outlet details alongside the agent's narrative description: *"Serendib Ocean Grill sits at the very edge of the seafront, where the sound of cutlery is accompanied by the rhythm of the Indian Ocean..."*

### Dietary Intelligence
The AI can filter and explain dietary options (Halal, Vegan, Gluten-Free) across all outlets — a critical feature for diverse international hotel guests. This is surfaced both conversationally and in the rendered cards.

---

## 8. Destination & Attraction Layer (Sri Lanka Experience)

### What It Is
Beyond the hotel, the platform becomes an expert on Sri Lanka itself. Attractions are stored as structured entities linked to each property.

### Attraction Data Model

| Field | Examples |
|---|---|
| `category` | Culture & Heritage, Wildlife, Beach, Adventure, Spiritual |
| `distance_km` | 2.4 |
| `travel_time_minutes` | 12 |
| `recommended_transport` | Tuk-Tuk, Shuttle, Walk |
| `best_time_to_visit` | Early Morning, Sunset, Anytime |
| `opening_hours` | 06:00 AM – 06:00 PM |
| `google_maps_url` | Direct map link |
| `description` | Full narrative including hidden tips |

### The AttractionCardUI
When a guest asks "What can I do near the hotel?", the agent calls `get_destination_attractions` and renders a card grid showing each attraction with distance, transport recommendation, best visiting time, and a Google Maps link — all within the chat window.

### Why It Differentiates
Most hotel chatbots only answer questions about the hotel. This platform extends into destination storytelling:
- Hidden cafés that locals love
- Best time to watch sunrise from Galle Fort ramparts
- Which beach cove is quiet in the morning
- Whale watching season specifics

This transforms the AI from a hotel FAQ system into a **trusted local travel companion**.

---

## 9. AI Itinerary Generator

### What It Is
A personalised multi-day stay planner that synthesises hotel outlets, partner experiences, and destination attractions into a beautifully rendered itinerary.

### Input Parameters

| Parameter | Options |
|---|---|
| `durationDays` | 1–7 days |
| `travelStyle` | `romantic`, `family`, `active`, `wellness`, `luxury` |
| `interests` | surfing, spa, history, wildlife, photography, tea tasting... |
| `experiencePreference` | `balanced`, `hotel_only`, `partner_only`, `destination_only` |
| `excludeCategories` | nightlife, adventure, cultural... |
| `maxPrice` | Per-activity budget cap |

### Three Experience Sources Woven Together

| Source Type | Badge | Example |
|---|---|---|
| `hotel_outlet` | **Hotel Featured** | Sunrise breakfast at Spices of Galle |
| `destination_attraction` | **Must See** | Galle Fort UNESCO Heritage Walk |
| `partner_experience` | **10% Guest Discount** | Ceylon Tea Estate private tasting |

### Time-of-Day Intelligence
The Itinerary sub-agent applies hard scheduling logic:
- **Early morning (06–09 AM):** Whale safaris, sunrise walks, ocean breakfast
- **Mid-day heat (13–16):** Spa, indoor tea tasting, villa quietude
- **Golden hour (17:30–21:30):** Sunset cocktails, candlelight dinner, stargazing

### The ItineraryCardUI
Each day is rendered as an expandable card showing:
- Time slot and activity name
- Source badge (Hotel / Must See / Partner Discount)
- Duration and price information
- Category icon and description

The agent's text response wraps this card in 1–2 sentences — keeping the visual UI as the hero.

---

## 10. Hour-by-Hour Resort Experience Timeline

### What It Is
A sensory simulation of an entire day at the hotel — helping guests visualise their stay before booking.

### Sample Timeline Slots

| Time | Title | Category | Sensory Highlight |
|---|---|---|---|
| 06:00 AM | Ocean Sunrise & Calm Wave Watch | `sensory_sunrise` | Gentle reef break before the resort wakes up |
| 08:00 AM | Organic Tropical Breakfast | `dining` | Cardamom, freshly baked hoppers, roasted coffee |
| 11:00 AM | Poolside Cabana Relaxation | `pool_quiet` | Cool ocean breezes under coconut palm shade |
| 04:00 PM | Ceylon Afternoon High Tea | `wellness` | Delicate porcelain clinks and warm cinnamon |
| 05:30 PM | Golden Hour Sunset Cocktail Soiree | `sunset_cocktail` | Crimson skies, crisp sea spray, acoustic guitar |
| 09:00 PM | Stargazing Candlelight Dinner | `dining` | Tiki torch glow, rhythm of night tides |

### Why It Converts
A guest who can *imagine* their morning — hearing the waves from their balcony, tasting freshly cracked coconut — is far more likely to book than one who read room dimensions. This timeline directly answers the emotional questions that drive booking decisions.

---

## 11. Memory Engine — Dual-Tier Guest Intelligence

### What It Is
A high-performance memory architecture that makes every guest feel remembered.

### Memory Tier 1 — Short-Term Session Memory (In-Memory, < 1ms)
**Scope:** Single conversation thread
**Storage:** In-memory map (not database — zero latency)
**Contents:**
- `dietaryTags` — ["Vegan", "Halal"]
- `preferredView` — "Ocean View"
- `bedType` — "King"
- `travelPartySize` — 2
- `travelDates` — check-in / check-out
- `budgetRange` — per-night comfort zone
- `activityInterests` — ["Spa", "Whale Watching"]
- `specialRequests` — "Anniversary surprise setup"
- `personalitySummary` — LLM-synthesised 1–2 sentence traveller persona

### Memory Tier 2 — Long-Term Guest Profile (Persistent, Cross-Session)
**Scope:** All sessions for a returning guest
**Storage:** PostgreSQL guest table
**Contents:** Historical preferences, past stays, dietary needs, accessibility requirements

### Zero-Latency Architecture
The platform never delays a response to extract memory. The flow is:
1. Chat request arrives → session memory is injected into agent prompt synchronously (< 1ms)
2. Agent streams response to guest
3. **After stream completes** → background worker analyzes conversation, extracts new preferences, updates memory

### Anonymous-to-Guest Binding
When an anonymous session guest provides their identity (name, email, phone), their accumulated session preferences automatically promote into a permanent guest profile — creating a continuity of care across future visits.

### How Memory Powers Storytelling
When a returning guest mentions they loved the spa last time, the agent already knows — and leads the conversation there. When a honeymooning couple is detected from conversation, the Itinerary agent skews toward romantic sunset dinners and private beach experiences.

---

## 12. Bot Persona Configuration Studio

### What It Is
A hotel-facing control panel where property managers customise the AI concierge's personality, capabilities, and widget appearance without touching code.

### Persona Configuration Options

| Setting | Options | Effect |
|---|---|---|
| `persona_preset` | `luxury`, `island_host`, `heritage`, `modern` | Base personality archetype |
| `personality_traits` | Confident, Friendly, Empathetic, Luxury, Warm, Storytelling | Multi-select trait combination |
| `response_detail` | `concise`, `balanced`, `rich` | Story depth and answer length |
| `greeting_style` | `standard`, `ayubowan` (Sri Lankan), `formal` | Opening message tone |
| `guardrail_strictness` | `standard`, `strict` | Content filtering intensity |

### Feature Capability Toggles

Each capability can be independently enabled or disabled per property:
- `enable_booking` — Activate/deactivate the Booking sub-agent
- `enable_dining` — Activate/deactivate the Dining & Spa sub-agent
- `enable_attractions` — Enable/disable destination recommendations
- `enable_stories` — Enable/disable heritage storytelling

### Widget Branding

| Setting | Description |
|---|---|
| `widget_primary_color` | Hex color for the chat widget theme |
| `widget_welcome_message` | First greeting when widget opens |
| `widget_suggestions` | Quick-reply suggestion chips (JSON array) |
| `widget_logo_url` | Hotel logo shown in widget header |
| `concierge_name` | The AI's name (e.g. "Aria", "Kasun", "Priya") |

### Caching Architecture
Bot config is cached in-memory with a **3-second TTL**. Changes made in the dashboard propagate to live conversations in under 3 seconds — no server restart required. If the API is unreachable, the system falls back to the previously cached config.

---

## 13. Property Management System (PMS Backend)

### What It Is
The Go REST backend serves as the authoritative data store for all hotel entities, structured as a multi-tenant property system.

### Core Entity Hierarchy

```
Property
├── Room Types (sensory descriptions, amenities, pricing, bed/view type)
├── Dining Outlets (cuisine, dietary tags, operating hours, dress code)
├── Media Assets (360 tours, drone clips, photos, floor plans, videos)
├── Heritage Stories (hotel history, legends, chef stories, sustainability)
├── Destination Attractions (Sri Lanka experience layer)
├── Seasonal Intelligence Guides (monsoon, whale watching, festivals)
├── Property Events (weddings, conferences, local festivals, specials)
└── Bot Configuration (persona, capabilities, widget branding)
```

### Data-to-Story Pipeline
Every entity created or updated in the PMS automatically triggers the RAG Knowledge Engine — the hotel's data instantly becomes part of the AI's conversational knowledge, no manual re-indexing required.

### PMS Adapter Pattern
The backend includes a PMS adapter interface and factory pattern, designed to plug into external Property Management Systems (OPERA, Cloudbeds, Mews, etc.) or the built-in internal adapter. This allows hotels already using an existing PMS to connect their live inventory without data duplication.

---

## 14. Seasonal Intelligence Layer

### What It Is
A calendar-aware knowledge layer that makes the AI's recommendations contextually relevant to the time of year.

### What Seasonal Guides Cover
- Monsoon season dates and what they mean for the guest experience
- Whale watching windows (blue whale, sperm whale, dolphin pods)
- Surfing swell conditions by month
- Turtle nesting periods on local beaches
- Cultural festival dates (Vesak, Perahera, Sinhala New Year)
- Holiday peak periods and expected crowd levels
- Sunrise and sunset time changes through the year

### How It Connects to Pricing
The Booking sub-agent uses seasonal data to explain pricing contextually:
> *"December is peak season here — whale watching is exceptional and the sea is perfectly calm. That's why nightly rates are higher. If you're flexible and can arrive a week later, you'll find the experience nearly identical at a lower rate."*

### Auto-Sync to Knowledge Base
Every seasonal guide created or updated in the dashboard is automatically embedded into Qdrant, making the AI aware of current seasonal conditions without any manual intervention.

---

## 15. Heritage Story Builder

### What It Is
A content management layer for the hotel's narrative identity — the stories that make a property unique.

### Story Categories
- History of the property (founding, architecture, ownership)
- Local legends and folklore connected to the location
- Chef and culinary heritage stories
- Sustainability initiatives and conservation efforts
- Nearby cultural traditions and indigenous knowledge
- Famous guests and memorable moments

### How Stories Enter Conversations
Heritage stories are embedded into Qdrant as knowledge chunks. When a guest asks anything that touches on history, culture, authenticity, or the "story behind" a place, the Concierge sub-agent retrieves relevant stories via semantic search and weaves them into responses:

> *"The property was built on land where a Dutch trading post once stood. The original coral stone walls are still visible in the eastern wing — a detail many guests discover on their morning walk before breakfast."*

This transforms the AI from an information retrieval system into a **living storyteller** for the property.

---

## 16. Chat Gateway & SSE Streaming

### What It Is
The real-time communication layer connecting the guest-facing widget to the LangGraph multi-agent backend.

### SSE Event Stream Protocol

```
thinking          → Model's intermediate reasoning text (activity indicator)
tool_start        → "generate_itinerary" called (shows loading state)
tool_end          → Tool execution complete
ui_payload        → Structured card data (parsed by frontend to render rich UI)
token             → Streaming text characters (narrative text appears word-by-word)
subagent_handoff  → Active sub-agent changed mid-conversation (e.g. Concierge → Booking)
done              → Stream complete, thread ID confirmed
error             → Terminal error occurred (carries a message field)
[DONE]            → Connection close signal
```

See `chat_gateway_api.md` §1 for the full event type reference and example stream.

### Why SSE Over WebSockets
Server-Sent Events (SSE) are unidirectional and simpler — ideal for an LLM streaming use case where the server pushes tokens to the client. No bidirectional channel needed, lower overhead, native browser support, automatic reconnection.

### Chat History Persistence
Every conversation is persisted with full message history, allowing:
- Resuming a conversation on a different device
- Hotel staff reviewing guest conversations in the dashboard
- Analytics on conversation patterns and unanswered questions

### Guest Feedback Collection
After any conversation, guests can submit a 1–5 star rating and optional comment. This feeds the hotel's continuous improvement loop and identifies which responses need refinement.

---

## 17. Hotel Dashboard & Admin Workspace

### What It Is
The operational control centre for hotel property managers — a Next.js admin interface with a comprehensive set of management sections.

### Dashboard Sections

| Section | Purpose |
|---|---|
| **Overview** | Live metrics, recent conversations, quick stats |
| **Rooms** | Room type management with sensory experience editor |
| **Outlets** | Dining, spa, and facility management |
| **Stories** | Heritage story content management |
| **Attractions** | Destination attraction management |
| **Seasons** | Seasonal intelligence guide management |
| **Events** | Property events, promotions, festivals |
| **Knowledge** | RAG document upload (PDF, markdown, text) |
| **Media** | Virtual tour, photo, video, floor plan management |
| **Bot Config** | AI persona, widget branding, capability toggles |
| **Bookings** | Booking inquiry and reservation management |
| **Requests** | Guest request tracking |
| **Analytics** | Conversation analytics and performance metrics |
| **Telemetry** | Agent performance monitoring |
| **Playground** | Live AI testing sandbox for hotel staff |
| **Widget** | Embeddable chat widget preview and code generator |
| **Packages** | Promotional package management |
| **Glossary** | Hotel terminology and FAQ management |

### The Playground
Hotel staff can test the AI agent live — simulating guest conversations before deploying widget updates. This ensures every persona change or knowledge addition is verified before guests experience it.

---

## 18. Vercel Blob Cloud Storage & Media Streaming Architecture

### What It Is
An end-to-end luxury media upload and CDN delivery pipeline leveraging private Vercel Blob Storage (`store_UZOT6VqAx2I3coOb`) and client-side pre-processing.

### Key Capabilities
- **In-Browser WebP Canvas Compression**: High-resolution camera photos (PNG/JPG up to 15MB) are automatically resized (max 2048px @ 0.88 quality) directly in browser memory, reducing payload sizes by 85–90%.
- **Zero-Latency Optimistic Visual Previews**: Uses local blob URLs (`URL.createObjectURL`) to render instant image previews in < 1ms before network transmission.
- **Direct Server-Side Vercel Blob Storage**: Employs `@vercel/blob`'s `put()` with private access isolation, preventing unauthorized external modifications.
- **Edge CDN Streaming Proxy (`/api/blob/view`)**: Serves private storage assets seamlessly to browser `<img>` elements and widget instances while enforcing `Cache-Control: public, max-age=31536000, immutable` for global sub-second loads.
- **Universal Multi-Domain Integration**: Wired across Rooms, Media Library, Dining/Spa Outlets, Experiences, Local Attractions, Events, Stories, Bot Crests, and Property Branding.

---

## 19. Multi-Tenant Token Budgeting & Automated Monthly Reset

### What It Is
A multi-tenant token consumption and metering engine that isolates and audits compute costs per luxury property.

### Key Capabilities
- **Granular Turn & Generative UI Accounting**: Tracks prompt tokens, completion tokens, subagent tool dispatches, and Generative UI card renders (1,000 tokens/rendered card).
- **Automated 1st-of-the-Month Reset Engine**: Evaluates `TokenResetDate` on every lookup and turn execution. On the 1st of every calendar month at 00:00:00 UTC, the Go backend automatically zeroes `tokens_used_this_month`, restores `token_status` to `"active"`, and advances `TokenResetDate` to the 1st of the next month.
- **Multi-Level Threshold Warnings**: Dynamic alerts when tenants reach 80% (`warning_80`) and 100% (`exceeded`) of their monthly quota with real-time UI badges.
- **Subagent Load Distribution Breakdown**: Visual telemetry tracking compute across Concierge, Booking, Dining & Spa, and Itinerary agents.

---

## 20. Digital Concierge Studio 2.0 & Live Preview

### What It Is
An interactive design and configuration studio allowing hotel managers to preview, brand, and customize their guest-facing digital concierge with zero code.

### Key Capabilities
- **Bi-Directional Synchronized Live Canvas**: Real-time simulation of live hotel web pages with instant desktop and mobile viewport toggles.
- **Visual Framing & Crop Geometry**: Support for Circle, Square, Rounded (16px), Cover (Fill), and Contain (Fit) crop presets with top/center/bottom alignment.
- **Multi-Target Concierge Imagery**: Simultaneously controls Header Avatar, Welcome Hero Banner, Floating Launcher Bubble, and Popover profile artwork.
- **Accordion Architecture with Clean Initial States**: Intuitive collapsible sections that keep the workspace uncluttered upon initial arrival.

---

## 21. How All Features Complement One Another

### The Unified Guest Journey

```text
       Guest Visits Resort Website
                 |
         [Digital Concierge Widget]
                 |
      [Router] → Routes to Concierge Sub-Agent
                 |
      [Knowledge Engine (RAG)] → Retrieves Sensory Story
                 |
      [Storytelling Response] → "Around sunrise, you'll hear the waves..."
                 |
         Guest Asks About Rooms
                 |
      [Router] → Handoff to Booking Sub-Agent
                 |
      [check_room_availability] → [Sensory Room Carousel UI]
                 |
      [get_price_breakdown] → Seasonal Narrative ("Peak whale season")
                 |
      [create_room_hold] → 15-Minute Hold + Countdown Card
                 |
      [generate_payment_link] → Direct Checkout
                 |
      [Memory Engine] → Preferences Saved Async
                 |
      [Itinerary Agent] → Personalised Day Plan Generated
                 |
      [Experience Timeline] → 24-Hour Sensory Preview
                 |
      Guest Books. Guest Arrives. Guest Returns.
                 |
      [Long-Term Memory] → "Welcome back. The Ocean View Suite
                             has been reserved for you again."
```

### Key Complementary Relationships

| Feature A | Feature B | How They Work Together |
|---|---|---|
| **Knowledge Engine** | **Storytelling** | RAG retrieves factual data and sensory narratives simultaneously — the AI greets both head and heart |
| **Seasonal Intelligence** | **Booking Engine** | Seasonal data provides the *why* behind pricing — guests understand value, not just cost |
| **Memory Engine** | **Sub-Agent System** | Preferences extracted in session are injected into every sub-agent's prompt — the AI always knows who it's speaking to |
| **Generative UI** | **Storytelling** | Text and rich cards arrive together — the story creates desire, the card provides detail |
| **Heritage Stories** | **RAG Pipeline** | Stories are auto-embedded into vector search — every story becomes conversationally accessible without explicit configuration |
| **Itinerary Generator** | **Destination Attractions** | Itineraries pull from hotel, partner, and destination data to create holistic plans no single-source system could produce |
| **Guardrail System** | **Bot Persona** | Refusal responses carry the hotel's brand voice — even a rejection feels like exceptional hospitality |
| **Bot Config** | **All Sub-Agents** | Every agent inherits the same persona, tone, and capability settings — the concierge feels like one coherent person across all topics |
| **Auto-Sync** | **Dashboard Edits** | Any content change in the dashboard instantly propagates to live conversations — no technical steps required from hotel staff |
| **Experience Timeline** | **Itinerary Generator** | The timeline answers "what is a day like?" while the itinerary answers "what should I do for 3 days?" — complementary pre-booking tools |
| **Vercel Blob Storage** | **Concierge Studio & Cards** | Stores high-resolution resort assets with client WebP compression, serving fast imagery to widget cards |
| **Token Reset Engine** | **Multi-Tenant System** | Enforces fair monthly usage per hotel, resetting automatically on the 1st of every month |

---

*Document generated from full codebase analysis of `chatbot-demo-admin` (Next.js 16 + LangGraph + Gemini 2.5 Flash) and `chatbot-demo-api` (Go + PostgreSQL + Qdrant Cloud).*
*Last updated: August 2026*
