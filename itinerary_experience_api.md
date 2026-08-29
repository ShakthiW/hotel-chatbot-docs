# Experience Timeline & Itinerary Generator API Specification

The **Experience Timeline & Itinerary Generator API** powers the luxury resort experience layer, sensory day timeline simulations, multi-tenant partner program management, and guest-controlled trip itineraries.

---

## 1. Unified Experience Management API

Manage in-hotel activities, partner programs, and local destination sights/attractions for a specific property.

> **Note on response envelope**: this document's examples use `{success, data}`, matching
> `chat_gateway_api.md` and `live_chat_and_handover_websocket_api.md` — see the envelope note
> in `chat_gateway_api.md` for why this differs from other docs in this set.
>
> **Note on `source_type` vocabularies**: this document uses `source_type` for two genuinely
> different things. §1's `Experience.source_type` (`in_hotel` / `partner` / `local_attraction`)
> classifies how a stored `Experience` record originated. §2's itinerary-output day-activity
> `source_type` (`hotel_outlet` / `partner_experience` / `destination_attraction`) is a
> separate, presentation-only label on a generated itinerary item — and `destination_attraction`
> in particular can be sourced from a `PropertyAttraction` record (see
> `property_management_api.md` §F), not an `Experience` at all. They are not meant to be the
> same enum; don't expect `Experience.source_type` values to appear verbatim in itinerary output.

### 1.1 List Experiences
Retrieves all experiences, filtered by source type, category, or partner status.

* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/experiences`
* **Authentication**: Public.
* **Query Parameters**:
  * `source_type` (optional): `in_hotel`, `partner`, or `local_attraction`.
  * `category` (optional): `wellness`, `dining`, `nightlife`, `culture`, `adventure`, `ocean`, `family`.
  * `is_partner` (optional): `true` to list only third-party partner programs.

#### Example Response (`200 OK`)
```json
{
  "success": true,
  "data": [
    {
      "id": "29768386-a4b0-4420-aaae-a8724a03b137",
      "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
      "title": "Local Sunset Bicycle Excursion",
      "source_type": "partner",
      "partner_name": "Galle Pedal & Coast",
      "category": "adventure",
      "description": "Guided bicycle tour along Galle coast line with stop at local tea kiosk.",
      "duration_minutes": 120,
      "price_per_person": 35.00,
      "currency": "USD",
      "partner_discount_pct": 20.00,
      "commission_rate_pct": 10.00,
      "rating": 4.8,
      "priority_weight": 8,
      "is_featured": true,
      "is_halal_friendly": true,
      "is_vegan_friendly": true,
      "is_kid_friendly": true,
      "image_url": "https://images.unsplash.com/photo-1544197150-b99a580bb7a8",
      "created_at": "2026-08-06T15:18:58Z",
      "updated_at": "2026-08-06T15:18:58Z"
    }
  ]
}
```

---

### 1.2 Create Experience
Create a new in-hotel activity, partner program, or local attraction.

* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/experiences`
* **Authentication**: Required — staff JWT scoped to this property.

#### Example Request Body
```json
{
  "title": "Mirissa VIP Blue Whale Catamaran Cruise",
  "source_type": "partner",
  "partner_name": "Mirissa Ocean Safari Club",
  "category": "ocean",
  "description": "Catamaran cruise with expert marine biologist to spot blue whales and spinner dolphins off the southern coast.",
  "duration_minutes": 240,
  "price_per_person": 120.00,
  "currency": "USD",
  "partner_discount_pct": 15.0,
  "priority_weight": 9,
  "is_featured": true
}
```

#### Example Response (`201 Created`)
```json
{
  "success": true,
  "data": {
    "id": "e47b3190-2810-410a-81a1-3990812b189a",
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "title": "Mirissa VIP Blue Whale Catamaran Cruise",
    "source_type": "partner",
    "partner_name": "Mirissa Ocean Safari Club",
    "category": "ocean",
    "description": "Catamaran cruise with expert marine biologist to spot blue whales and spinner dolphins off the southern coast.",
    "duration_minutes": 240,
    "price_per_person": 120.00,
    "currency": "USD",
    "partner_discount_pct": 15.0,
    "priority_weight": 9,
    "is_featured": true
  }
}
```

---

### 1.3 Update Experience
* **HTTP Method**: `PUT`
* **Path**: `/api/v1/properties/{id}/experiences/{exp_id}`
* **Authentication**: Required — staff JWT scoped to this property.

### 1.4 Delete Experience
* **HTTP Method**: `DELETE`
* **Path**: `/api/v1/properties/{id}/experiences/{exp_id}`
* **Authentication**: Required — staff JWT scoped to this property.

### 1.5 Get One Experience
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/experiences/{exp_id}`
* **Authentication**: Public.

### 1.6 Record Experience Event (Guest Telemetry)
Records a guest-facing interaction (e.g. a view or click) with an experience card, for the
Experience Analytics dashboard.

* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/experiences/events`
* **Authentication**: Public.

### 1.7 Get Experience Analytics
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/experiences/analytics`
* **Authentication**: Required — staff JWT scoped to this property.

---

## 2. Multi-Day Itinerary Generator API

Generates tailored multi-day experience itineraries, combining tenant priority weights with guest preference overrides.

* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/itinerary/generate`
* **Authentication**: Public — called server-to-server by the guest chat agent during a live
  conversation.

### Request Payload Parameters
| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `duration_days` | Integer | Yes | Number of days for the itinerary (1 to 7). |
| `travel_style` | String | No | Travel style e.g. `romantic`, `family`, `active`, `wellness`, `luxury`. |
| `interests` | Array[String] | No | Specific interest tags e.g. `["surfing", "spa", "history"]`. |
| `experience_preference` | String | No | `balanced` (default), `hotel_only`, `partner_only`, `destination_only`. |
| `source_type_filter` | String | No | Guest source filter override e.g. `destination_only` (NO hotel promotion). |
| `exclude_categories` | Array[String] | No | Categories to exclude e.g. `["nightlife", "alcohol", "water_sports"]`. |
| `max_price_per_activity` | Float | No | Maximum price cap per activity item. |

#### Example Request
```json
{
  "duration_days": 3,
  "travel_style": "romantic",
  "interests": ["spa", "fine dining", "sunset"],
  "experience_preference": "balanced",
  "exclude_categories": ["nightlife"]
}
```

#### Example Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "property_name": "Grand Ocean Resort & Spa",
    "duration_days": 3,
    "travel_style": "romantic",
    "experience_preference": "balanced",
    "days": [
      {
        "day_number": 1,
        "title": "Day 1: Arrival, Coastal Sunset & Culinary Indulgence",
        "activities": [
          {
            "time_of_day": "Morning (08:00 AM)",
            "title": "Organic Tropical Breakfast & Sunrise Wave Watch",
            "source_type": "hotel_outlet",
            "category": "Breakfast & Dining",
            "description": "Enjoy freshly cracked coconuts, egg hoppers, and artisanal coffee at Serendib Ocean Grill.",
            "duration": "1.5 hrs",
            "price_info": "Included in Resort Package",
            "badge": "Hotel Featured"
          },
          {
            "time_of_day": "Afternoon (02:00 PM)",
            "title": "VIP Blue Whale & Catamaran Cruise (Mirissa Ocean Safari Club)",
            "source_type": "partner_experience",
            "category": "Adventure & Marine",
            "description": "Catamaran cruise with expert marine biologist to spot blue whales and spinner dolphins.",
            "duration": "240 mins",
            "price_info": "$120.00 USD",
            "badge": "Partner Rated 4.9★"
          },
          {
            "time_of_day": "Evening (06:00 PM)",
            "title": "Golden Hour Sunset Cocktails & Seafood Grill",
            "source_type": "hotel_outlet",
            "category": "Dining & Sunset",
            "description": "Watch the sky turn crimson over Galle Lighthouse while enjoying jumbo prawns at Cinnamon & Spice Tea Lounge.",
            "duration": "2 hrs",
            "price_info": "À la carte / Resort Credit Eligible",
            "badge": "Sunset Highlight"
          }
        ]
      }
    ]
  }
}
```

---

## 3. Hour-by-Hour Sensory Day Timeline API

Returns the 24-hour sensory day rhythm for the resort (from 06:00 AM Sunrise to 09:00 PM Stargazing).

* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/experience-timeline`
* **Authentication**: Public.

#### Example Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "property_name": "Grand Ocean Resort & Spa",
    "slots": [
      {
        "id": "41ebcc90-b0e4-4a0a-b74b-f1604f4effea",
        "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
        "time_slot": "06:00 AM",
        "title": "Sensory Ocean Sunrise & Calm Wave Watch",
        "category": "sensory_sunrise",
        "description": "Soft morning light illuminates the Indian Ocean. Perfect for verandah coffee and serene quietude.",
        "crowd_level": "Low (Quiet)",
        "sensory_highlight": "Listen to gentle reef break waves before the resort wakes up.",
        "sort_order": 1
      },
      {
        "id": "9e2e5060-716c-42c6-aa13-08622901e0e2",
        "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
        "time_slot": "08:00 AM",
        "title": "Organic Tropical Breakfast Buffet",
        "category": "dining",
        "description": "Live hopper stations, fresh papaya, king coconuts, and Ceylon tea at Spices of Galle.",
        "crowd_level": "Moderate",
        "sensory_highlight": "Fragrant cardamom, freshly baked hoppers, and roasted coffee aromas.",
        "sort_order": 2
      }
    ]
  }
}
```
