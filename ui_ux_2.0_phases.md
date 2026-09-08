# holaa UI/UX 2.0 — Generative UI Redesign Phases

> **Principle**: "Don't just answer the guest. Help them experience the answer."

This document tracks the complete 5-phase redesign of holaa's generative UI system, transforming the chat from functional widgets into a premium AI hospitality experience.

---

## Architecture Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Chat Targets** | Redesign both guest-facing AND admin-facing | Both experiences need premium quality |
| **Image Strategy** | AI-generated placeholders → Firebase later | Rapid prototyping, replaceable assets |
| **Backend APIs** | Full functional where possible, mock PMS | Simulate PMS with existing database layer |
| **Map Provider** | Google Maps Platform | Premium experience, rich SDK |
| **Real-time** | WebSocket subscriptions | ServiceRequest + Notifications need real push |
| **Room Detail UX** | Slide-over panel | Immersive without losing chat context |
| **CSS Framework** | Tailwind v4 (existing) | Extend with luxury design tokens |
| **Component Library** | shadcn/ui (existing) | Extend with custom generative UI primitives |

---

## Phase 1: Foundation & Design System + Core Widget Redesign ← CURRENT
**Status**: 🟡 In Progress  
**Impact**: Very High — delivers ~60% of visual transformation  
**Scope**: ~40 files modified/created

### 1.1 Design System Foundation
- [ ] Extend `globals.css` with widget design tokens, animations, skeleton utilities
- [ ] Create `WidgetShell.tsx` — shared wrapper for all generative UI widgets
- [ ] Create `WidgetSkeleton.tsx` — reusable skeleton primitives with shimmer
- [ ] Create `WidgetCTA.tsx` — structured action button with luxury styling
- [ ] Create `WidgetTag.tsx` — compact metadata tag/chip component
- [ ] Create `ImageCarousel.tsx` — touch-swipe image carousel with lazy loading
- [ ] Create `HolaaRecommendationBadge.tsx` — "holaa'S RECOMMENDATION" badge

### 1.2 Chat Architecture Refactor
- [ ] Decompose `ChatSimulator.tsx` (821 lines → modular components)
- [ ] Create `WidgetRenderer.tsx` — central widget dispatcher
- [ ] Create `ActionDispatcher.tsx` — structured action handler system
- [ ] Create `QuickActionRail.tsx` — contextual actions after responses
- [ ] Create `ContextualSuggestionUI.tsx` — natural next-question suggestions
- [ ] Remove hardcoded text-parsing fallbacks (~250 lines)
- [ ] Redesign message bubbles (ivory bg, warm typography)
- [ ] Redesign chat header (hospitality-branded, not developer-styled)
- [ ] Add progressive widget rendering (skeleton → populated → CTA active)

### 1.3 ThinkingAccordion → ConciergeActivityUI
- [ ] Replace raw chain-of-thought with human-friendly activity indicators
- [ ] States: queued → processing → completed → error
- [ ] Parse thinking into friendly labels
- [ ] Subtle animated progress indicators

### 1.4 RoomCarouselUI 2.0
- [ ] Image-first layout with hero photography
- [ ] Multi-image carousel per room
- [ ] Tag row (view, bed, guests, size)
- [ ] Sensory narrative description
- [ ] Availability indicator
- [x] holaa Recommendation badge — reused for a plain "Matches: ..." filter-match reason too,
      with its own label ("Matches Your Search") so a factual filter match doesn't borrow the
      "holaa's Recommendation" framing meant for genuine opinionated picks (2026-09-08)
- [x] "Explore Room" + "Compare" CTAs (2026-09-08)
- [ ] Snap-scroll with partial next-card visibility
- [ ] Skeleton loading state
- [x] RoomComparisonUI (2026-09-08) — side-by-side table, capped at 4 rooms rather than 3

### 1.5 DiningOutletUI 2.0
- [ ] Restaurant photography
- [ ] Price level, hours, dietary tags, availability, featured dish, rating
- [ ] holaa recommendation badge
- [ ] Dietary filter chips
- [ ] "View Menu" + "Reserve Table" CTAs

### 1.6 AttractionCardUI 2.0
- [ ] Attraction photography
- [ ] Weather suitability, cost, duration, crowd level, holaa score
- [ ] "View on Map" + "Add to Itinerary" + "Ask holaa" CTAs

### 1.7 BookingHoldCardUI 2.0
- [ ] Room image thumbnail
- [ ] Date/guest/rate breakdown
- [ ] Circular progress timer (clear but not stressful)
- [ ] Warm ivory/gold styling
- [ ] "Continue to Payment" + "Release Room" CTAs

---

## Phase 2: New Core Widgets
**Status**: 🟢 Completed  
**Impact**: High  
**Scope**: ~20 new files

### 2.1 RoomDetailUI
- [x] Slide-over panel with large image gallery + video
- [x] Floor plan, amenities grid, guest reviews
- [x] Available dates, price, policies
- [x] AI-generated atmosphere description
- [x] "Check Availability" / "Hold This Room" / "Ask About This Room"

### 2.2 ExperienceGalleryUI
- [x] Media-first widget for visual questions
- [x] Large featured image/video, horizontal thumbnails
- [x] Experience label, time of day, AI narration
- [x] "Ask about this experience" CTA

### 2.3 DiningMenuUI
- [x] Visual menu with dish photos
- [x] Name, description, price, dietary tags, spice level, chef recommendation
- [x] "Add to Room Service" / "Reserve Table"

### 2.4 WeatherExperienceUI
- [x] Premium compact weather card
- [x] Temperature, conditions, rain %, wind, humidity, sunrise/sunset, sea
- [x] "How this affects your stay" — AI narrative connecting weather to experience

### 2.5 GuestPreferenceUI
- [x] Lightweight selectable chips (Quiet mornings, Ocean views, Food, etc.)
- [x] Persist selections in conversation context
- [x] Elegant, non-intrusive form alternative

### 2.6 PersonalRecommendationUI
- [x] "Based on what you've told me…" card
- [x] Room/restaurant/activity recommendation with reasoning
- [x] Uses collected guest preferences

### 2.7 BookingConfirmationUI
- [x] "Your stay is confirmed" celebration card
- [x] Booking ref, guest name, hotel, room, dates, total, QR code
- [x] "View Reservation" / "Add to Calendar" / "Message Hotel" / "Plan Your Stay"

### 2.8 PaymentCheckoutCardUI 2.0
- [x] Hotel logo, room image, reservation details, cancellation policy
- [x] Trustworthy premium styling (not dark gradient)
- [x] "Complete Reservation" CTA
- [x] On success → render BookingConfirmationUI

---

## Phase 3: Travel & Transport + Itinerary Redesign
**Status**: ⬜ Not Started  
**Impact**: Medium-High  
**Scope**: ~15 files

### 3.1 DestinationMapUI
- [ ] Google Maps interactive map
- [ ] Category markers (Hotel, Beaches, Restaurants, Attractions, Transport, etc.)
- [ ] Filter by category, travel time, directions
- [ ] Click markers to open attraction cards
- [ ] Mobile-friendly zoom/pan

### 3.2 AirportTransferUI
- [ ] Airport ↔ Hotel transfer booking
- [ ] Vehicle options (Car, Van, Luxury, Child seat)
- [ ] Distance, time, price, passengers, luggage
- [ ] "Book Transfer" CTA

### 3.3 LocalTransportUI
- [ ] "How do I get there?" multi-mode transport options
- [ ] Tuk-tuk, Taxi, Hotel shuttle, Train, Bus, Walking, Rental car
- [ ] Time, distance, cost, comfort, recommended option

### 3.4 ItineraryCardUI 2.0 (merge into TimelineJourney)
- [ ] Elegant Day tabs with Morning/Afternoon/Evening sections
- [ ] Activity cards with time, duration, distance, cost, transport, image
- [ ] Drag/reorder/remove capabilities
- [ ] AI replanning triggers ("Make it more relaxed", "Rainy day version", etc.)

### 3.5 TimelineJourneyCardUI 2.0
- [ ] Luxury travel journal aesthetic
- [ ] Experience photos, short stories, mood indicators
- [ ] Horizontal swipe on mobile

### 3.6 ExperienceTimelineUI 2.0
- [ ] Background imagery per time slot
- [ ] Ambient description, temperature, suggested activity
- [ ] Heavy imagery for mental pre-experience

---

## Phase 4: Guest Journey & Lifecycle Widgets
**Status**: ⬜ Not Started  
**Impact**: Medium  
**Scope**: ~15 files

### 4.1 StayCompanionUI
- [ ] Post-booking lifecycle hub
- [ ] Replace discovery CTAs with stay-prep actions
- [ ] Reservation summary, weather, upcoming activities

### 4.2 ServiceRequestUI
- [ ] In-stay service requests (towels, housekeeping, room service, etc.)
- [ ] Request ID, status tracking, estimated completion
- [ ] WebSocket-powered real-time status updates

### 4.3 NotificationCardUI
- [ ] Proactive contextual notifications via WebSocket
- [ ] Weather alerts, transfer reminders, sunset times, activity departures
- [ ] Subtle, non-intrusive card styling

### 4.4 HumanHandoffUI
- [ ] Graceful AI → human agent transition
- [ ] Agent availability, estimated response time, department
- [ ] Context preservation (guest doesn't repeat question)

### 4.5 GuestReviewUI
- [ ] Verified guest review quotes with ratings
- [ ] Guest type, date, highlighted context
- [ ] Aggregate mention counts

### 4.6 ReviewSummaryUI
- [ ] Overall rating, sentiment analysis, top positives, common complaints
- [ ] "Best suited for" and "Potential drawbacks"
- [ ] Balanced, trustworthy presentation

### 4.7 OfferCardUI
- [ ] Contextual promotions (not aggressive upselling)
- [ ] Valid dates, inclusions, original/discounted price, savings
- [ ] "Apply Offer" CTA

### 4.8 UpsellExperienceUI
- [ ] Experience-driven contextual upselling
- [ ] "What would make your stay more special?"
- [ ] Room upgrades, spa, excursions, private dining

---

## Phase 5: Master Composites & Polish
**Status**: ⬜ Not Started  
**Impact**: Medium  
**Scope**: ~12 files

### 5.1 ExperienceComposerUI
- [ ] Higher-level composition layer for "experience bundles"
- [ ] Renders image + weather + story + room + availability + CTA as composite

### 5.2 BookingJourneyUI
- [ ] Multi-step conversational booking flow
- [ ] Dates → Room → Extras → Review → Hold → Payment → Confirmation
- [ ] Step indicator, never leaves the conversation

### 5.3 StayPlannerUI
- [ ] Complete digital stay companion
- [ ] Tabs: Reservation, Transfer, Restaurants, Activities, Weather, Itinerary, Requests

### 5.4 SourceTrustUI
- [ ] Subtle source attribution for factual answers
- [ ] "Based on hotel's current guest policy — Updated 2 days ago"

### 5.5 Comprehensive Type System
- [ ] `src/types/widget-types.ts` — all widget payload interfaces
- [ ] Action type definitions
- [ ] Guest journey state enum (DISCOVER → EXPLORE → COMPARE → BOOK → PREPARE → STAY → RETURN)
- [ ] Widget lifecycle state enum (idle → loading → streaming → ready → selected → submitted → success → error → expired → disabled)

### 5.6 Backend Tool Extensions
- [ ] Extend tool payloads with images, amenities, availability, suggestedActions
- [ ] New tools: weather, service requests, airport transfer, dining menu
- [ ] All tools emit `suggestedActions` in `__UI__` payloads

### 5.7 WebSocket Infrastructure
- [ ] WebSocket server for real-time notifications
- [ ] Service request status updates
- [ ] Proactive notification delivery
- [ ] Connection management and reconnection logic

### 5.8 Performance & Accessibility Audit
- [ ] Lazy image loading across all widgets
- [ ] Responsive image sizing
- [ ] Virtualized long lists
- [ ] ARIA labels, focus states, keyboard navigation
- [ ] `prefers-reduced-motion` support
- [ ] Screen reader compatibility

---

## Guest Journey State Machine

```
DISCOVER → EXPLORE → COMPARE → BOOK → PREPARE → STAY → RETURN
    │          │          │        │        │        │       │
    ▼          ▼          ▼        ▼        ▼        ▼       ▼
  General    Rooms     Compare  Hold +   Transfer  Service  Reviews
  inquiry    Dining    Rooms    Payment  Dinner    Requests Rebooking
  Attract.   Attract.  Prices   Confirm  Weather   Notifs   Offers
```

Widget CTAs and suggestions adapt based on current journey stage.

---

## SSE Protocol Extensions (Planned)

Current event types (verified against `src/app/api/chat/route.ts`):
- `token` — streamed text
- `thinking` — AI reasoning (will become ConciergeActivityUI)
- `ui_payload` — structured widget data
- `tool_start` / `tool_end` — tool execution lifecycle
- `subagent_handoff` — subagent transition. This note used to flag a real mismatch: the
  server emitted `subagent_handoff` but `ChatSimulator.tsx` listened for `agent_handoff`, so
  the handoff banner in the admin preview never fired. Fixed 2026-09-07 by making the
  listener match what's actually emitted, rather than the other way around, since
  `subagent_handoff` is the name consistent with how the rest of this codebase refers to
  concierge/booking/dining_spa/itinerary as "subagents."
- `done` — stream complete
- `error` — error

New event types needed:
- `suggested_actions` — contextual next actions
- `journey_state` — current guest journey phase
- `widget_skeleton` — early skeleton trigger before full payload
- `notification` — proactive push notification (via WebSocket)
- `service_update` — real-time service request status (via WebSocket)

---

## File Structure (Target)

```
src/components/generative-ui/
├── shared/
│   ├── WidgetShell.tsx
│   ├── WidgetSkeleton.tsx
│   ├── WidgetCTA.tsx
│   ├── WidgetTag.tsx
│   ├── ImageCarousel.tsx
│   └── HolaaRecommendationBadge.tsx
├── RoomCarouselUI.tsx          ← Phase 1 (redesign)
├── RoomComparisonUI.tsx        ← Phase 1 (new)
├── RoomDetailUI.tsx            ← Phase 2
├── DiningOutletUI.tsx          ← Phase 1 (redesign)
├── DiningMenuUI.tsx            ← Phase 2
├── AttractionCardUI.tsx        ← Phase 1 (redesign)
├── BookingHoldCardUI.tsx       ← Phase 1 (redesign)
├── PaymentCheckoutCardUI.tsx   ← Phase 2 (redesign)
├── BookingConfirmationUI.tsx   ← Phase 2
├── ConciergeActivityUI.tsx     ← Phase 1 (replaces ThinkingAccordion)
├── ItineraryCardUI.tsx         ← Phase 3 (redesign)
├── TimelineJourneyCardUI.tsx   ← Phase 3 (redesign)
├── ExperienceTimelineUI.tsx    ← Phase 3 (redesign)
├── ExperienceGalleryUI.tsx     ← Phase 2
├── WeatherExperienceUI.tsx     ← Phase 2
├── GuestPreferenceUI.tsx       ← Phase 2
├── PersonalRecommendationUI.tsx ← Phase 2
├── DestinationMapUI.tsx        ← Phase 3
├── AirportTransferUI.tsx       ← Phase 3
├── LocalTransportUI.tsx        ← Phase 3
├── StayCompanionUI.tsx         ← Phase 4
├── ServiceRequestUI.tsx        ← Phase 4
├── NotificationCardUI.tsx      ← Phase 4
├── HumanHandoffUI.tsx          ← Phase 4
├── GuestReviewUI.tsx           ← Phase 4
├── ReviewSummaryUI.tsx         ← Phase 4
├── OfferCardUI.tsx             ← Phase 4
├── UpsellExperienceUI.tsx      ← Phase 4
├── ExperienceComposerUI.tsx    ← Phase 5
├── BookingJourneyUI.tsx        ← Phase 5
├── StayPlannerUI.tsx           ← Phase 5
├── SourceTrustUI.tsx           ← Phase 5
└── QuickActionRail.tsx         ← Phase 1
```

---

## 2026-09-08 Addendum: Filtering, Security & Interaction Layer

A separate initiative, run alongside this component-building roadmap rather than as one of
its numbered phases — the "Generative UI Blueprint," which audited how the generative UI
system actually behaved in production (not just which components existed) and shipped fixes
across four phases. Full detail lives in `booking_engine_api.md`, `chat_gateway_api.md`, and
`widget_embed_architecture.md`; this is the summary for readers tracking overall UI progress
from this document.

**Phase 0 — Security & correctness, no visible behavior change:**
- Deleted a dead, unused second Go chat pipeline (`completions_handler.go`) that had no caller
- Closed a cross-tenant data risk: every LangGraph tool now trusts the request-verified
  `configurable.propertyId` over its own optional, LLM-suppliable argument
- Fixed an XSS gap in `widget.js`'s card renderer — tool/RAG-derived text was reaching
  `innerHTML` unescaped, contrary to the file's own documented policy
- Fixed the `subagent_handoff`/`agent_handoff` mismatch noted above
- Found and fixed a real production bug along the way, not originally in scope: the Go
  backend's rate limiter was wrapped *outside* CORS, so a rate-limited response (including a
  CORS preflight) carried no `Access-Control-Allow-Origin` header at all — browsers report
  that as a generic network failure with the real 429 status invisible to the caller. Reordered
  so CORS always wraps rate limiting, and raised the default limit (2 req/sec → 10 req/sec) to
  match what one ordinary dashboard session actually needs.

**Phase 1 — Real filtering + empty state:** see `booking_engine_api.md` §1's hard-filter and
`no_match_context` documentation above.

**Phase 2 — Guest/admin parity:** `widget.js` gained the 5 card types it was previously
missing (`attractions`, `payment_link`, `experience_timeline`, `preferences`, `human_handover`)
and wired up room hero images via its own `safeImageUrl()`, which existed but was never called.

**Phase 3 — Enhanced catalog:** comparison table, "Matches: ..." reasoning tags, real
"Only N left" scarcity badges, per-card quick actions, live re-ranking (see
`platform_features.md` §5), and a first accessibility pass — the chat launcher across all
three surfaces was a `<div>` with a click handler and no keyboard affordance at all;
role/tabindex/keydown handling, focus management, `aria-label`s, a `role="log"` live region on
the message list, and `prefers-reduced-motion` support were added. Item 5.8 below ("Performance
& Accessibility Audit") is still open as a fuller pass — this was scoped to the chat surfaces
specifically, not the whole dashboard.

---

*Last updated: 2026-09-08*
