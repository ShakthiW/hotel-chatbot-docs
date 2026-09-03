# Omnichannel Booking Destinations & Distribution Architecture

This document specifies the **holaa V1 Omnichannel Booking Destination Architecture**, URL resolution priority, parameter injection, room-level override hierarchy, outbound intent tracking, and safe AI conversation guardrails.

---

## 1. Overview & Architectural Principles

In holaa V1, rather than attempting direct two-way PMS inventory synchronization, the platform operates in **External Booking Engine Mode** (`booking_mode: "external"`). This enables luxury resort operators to direct guests to their existing high-converting direct booking engine or preferred OTA distribution channels with zero technical friction.

### Supported Channels
1. **Direct Hotel Website (`direct`)**: Highest margin (0% OTA commission).
2. **Booking.com (`booking_com`)**: Global OTA distribution with automatic affiliate slug fallback.
3. **Agoda (`agoda`)**: Dominant Asia-Pacific OTA metasearch.
4. **Expedia (`expedia`)**: Global Americas & European distribution.
5. **Other Custom Portal (`other`)**: Any bespoke booking engine, reservation widget, or partner URL.

---

## 2. Destination Resolution Precedence Hierarchy

When a guest requests room availability or booking assistance, the backend [`destination_resolver.go`](file:///Users/shakthirw/Downloads/Chatbot%20Demo/chatbot-demo-api/pkg/channels/destination_resolver.go) resolves the primary action and secondary channels according to strict 4-tier precedence:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Specific Room Override (Direct, Booking.com, Agoda, etc.)│ ◄── Tier 1 (Top Priority)
└──────────────────────────────┬──────────────────────────────┘
                               │ (if room has no override)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Property-Level Configured Destinations (Integrations Hub)│ ◄── Tier 2 (Preferred Channel)
└──────────────────────────────┬──────────────────────────────┘
                               │ (if no destination configured)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Legacy Automatic Booking.com Deep-Link Engine            │ ◄── Tier 3 (Universal Fallback)
└──────────────────────────────┬──────────────────────────────┘
                               │ (if property has no slug)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. "Inquire About This Room" Modal Fallback                 │ ◄── Tier 4 (Zero Link State)
└─────────────────────────────────────────────────────────────┘
```

### Precedence Rules:
1. **Tier 1 (Room Override)**: If a specific room category has a custom booking URL configured (e.g. `/book/ocean-view-suite`), it takes precedence over property defaults (`source: "room_override"`).
2. **Tier 2 (Property Preferred)**: If no room override exists, holaa uses the property's configured preferred channel (e.g. `direct` with URL `https://hotel.com/reservations`, `source: "property_default"`).
3. **Tier 3 (Legacy Auto Booking.com)**: If the property has not configured any explicit destinations, holaa generates a pre-filled deep-link using the hotel's `booking_slug` and affiliate AID (`source: "legacy_auto"`).
4. **Tier 4 (Inquiry Fallback)**: If no URL can be constructed, holaa renders an **"Inquire About This Room"** button (`inquiry_fallback: true`) opening the luxury pre-filled `RoomInquiryModal`.

---

## 3. Dynamic Query Parameter Injection

The resolver inspects the target destination URL and intelligently appends guest check-in, check-out, and occupancy parameters without corrupting existing query strings:

| Channel | Parameter Schema | Example Appended URL |
| :--- | :--- | :--- |
| **Booking.com** | `checkin=YYYY-MM-DD`<br>`checkout=YYYY-MM-DD`<br>`group_adults=N`<br>`aid=304142` | `https://www.booking.com/hotel/lk/hotel.html?aid=304142&checkin=2026-09-01&checkout=2026-09-05&group_adults=2` |
| **Agoda** | `checkin=YYYY-MM-DD`<br>`checkout=YYYY-MM-DD`<br>`group_adults=N` | `https://www.agoda.com/hotel.html?checkin=2026-09-01&checkout=2026-09-05&group_adults=2` |
| **Direct & Others** | `checkin=YYYY-MM-DD`<br>`checkout=YYYY-MM-DD`<br>`group_adults=N` | `https://yourhotel.com/book?checkin=2026-09-01&checkout=2026-09-05&group_adults=2` |

---

## 4. UI Rendering & Generative Room Cards

The Generative UI (`RoomCarouselUI.tsx` and `RoomDetailUI.tsx`) renders dynamic CTAs:

```
┌─────────────────────────────────────────────────────────────┐
│ [ ROOM CARD ]                                               │
│ Ocean View Suite • 75 sqm • King Bed                        │
│ $450 / night • $1,800 Total (4 Nights)                      │
│                                                             │
│ ┌───────────────────────────┐ ┌────────┐ ┌────────────────┐ │
│ │  Book Direct (Primary)    │ │ More ▾ │ │  Details ➔     │ │
│ └───────────────────────────┘ └────────┘ └────────────────┘ │
│                                  │                          │
│                                  ▼                          │
│                      ┌──────────────────────┐               │
│                      │ • Book on Booking.com│               │
│                      │ • Book on Agoda      │               │
│                      │ • Send Direct Inquiry│               │
│                      └──────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

- **Primary Button**: Uses the hotel's configured preferred channel label (`"Book Direct"`, `"Book on Booking.com"`, `"Book on Agoda"`, etc.).
- **Multi-Channel Popover**: When multiple secondary channels are configured, guests can expand the dropdown to choose alternative booking portals.
- **Inquiry Trigger**: If the destination is missing or unverified, the CTA displays `"Inquire About This Room"`.

---

## 5. Outbound Click Tracking & Intent Analytics

To provide transparency into how AI conversations drive revenue without falsely claiming live PMS integration, holaa tracks **Outbound Booking Intent**:

### Tracked Metadata:
- `property_id`: Target hotel UUID
- `room_id` & `room_code`: Specific category clicked (`OVS-01`)
- `channel`: Destination channel (`direct`, `booking_com`, `agoda`, `expedia`, `other`)
- `destination_url`: Exact outbound URL
- `source`: Resolution tier (`manual`, `room_override`, `property_default`, `legacy_auto`)
- `session_id`: Conversational thread ID
- `created_at`: Exact ISO-8601 timestamp

### Dashboard Terminology Rules:
> [!IMPORTANT]
> Because live PMS holds are not connected in V1 External Mode, dashboards MUST use the following accurate terminology:
> - **Booking Intent Clicks** (NOT "Confirmed Bookings")
> - **Estimated Room Views** (Impressions inside chat widgets)
> - **Booking CTA CTR %** (Click-Through Rate: Clicks / Impressions)
> - **Top Destination Channel** (Most frequently chosen provider)

---

## 6. Safe Hospitality AI Language Guardrails

System prompts ([`defaultPrompts.ts`](file:///Users/shakthirw/Downloads/Chatbot%20Demo/chatbot-demo-admin/src/lib/agent/prompts/defaultPrompts.ts)) strictly enforce that the AI concierge:
1. **Never claims unverified live PMS holds**: Do not state *"I have reserved this room for you"*, *"3 rooms remain"*, or *"Your booking is 100% confirmed"*.
2. **Conveys sensory & emotional room storytelling**: Emphasizes door-opening sensations, views, and dimensions.
3. **Presents exact calculated rates**: Computes total price for requested stay duration with currency.
4. **Guides guests safely to external checkout**: States *"You can complete your reservation directly on the official hotel booking engine or via your preferred booking platform below."*
