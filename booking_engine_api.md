# Conversational Booking & Availability Engine API Specification

The **Conversational Booking & Availability Engine API** provides room inventory checking, price breakdown calculation, 15-minute temporary booking holds, secure payment checkout link generation, **omnichannel booking destination management**, and **outbound booking click & intent analytics** for the AI Hospitality Platform.

---

## Endpoints Overview

| Method | Endpoint | Auth | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/properties/{id}/booking/check-availability` | Public | Checks live room category availability, nightly rates, and resolves dynamic booking channel actions. |
| `POST` | `/api/v1/properties/{id}/booking/price-breakdown` | Public | Calculates itemized taxes, service charges, and resort credits. |
| `POST` | `/api/v1/properties/{id}/booking/hold` | Public | Creates a 15-minute temporary room hold reservation (Integrated/Simulator Mode). |
| `POST` | `/api/v1/properties/{id}/booking/payment-link` | Public | Generates a payment link and checkout UI payload. |
| `GET` | `/api/v1/properties/{id}/booking/destinations` | Public | Retrieves property-level configured booking channels, preferred channel, room override status, and setup health. |
| `PUT` | `/api/v1/properties/{id}/booking/destinations` | Staff | Updates property-level booking destinations, preferred channel, and booking mode. |
| `POST` | `/api/v1/properties/{id}/booking/destinations/test` | Staff | Validates a destination URL and updates the `last_verified_at` timestamp. |
| `PUT` | `/api/v1/properties/{id}/rooms/{roomId}/booking-destinations` | Staff | Sets or clears room-specific booking destination overrides. |
| `POST` | `/api/v1/properties/{id}/booking/track-click` | Public | Asynchronously logs guest outbound booking intent clicks with channel attribution. |
| `GET` | `/api/v1/properties/{id}/booking/analytics` | Staff | Aggregates outbound booking clicks, room CTR, channel distribution, and daily intent trends. |

The four `check-availability`/`price-breakdown`/`hold`/`payment-link`/`track-click` endpoints
and the `GET` on `destinations` are public because they're invoked server-to-server by the
guest chat backend on behalf of an unauthenticated guest — there is no logged-in user at that
point in the conversation. Everything that configures or reports on a property (`PUT`/`test`
destinations, room overrides, analytics) requires a staff JWT.

---

## 1. Check Room Availability & Resolve Booking Destinations
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/check-availability`
* **Authentication**: Public.

Checks room availability for specified dates and party size, and automatically resolves dynamic booking actions based on the 4-tier precedence engine (**Room Override** $\rightarrow$ **Property Default** $\rightarrow$ **Legacy Auto Booking.com** $\rightarrow$ **Inquiry Fallback**).

### Request Body
```json
{
  "check_in": "2026-09-01",
  "check_out": "2026-09-05",
  "adults": 2,
  "children": 0
}
```

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Room availability checked successfully",
  "data": {
    "check_in": "2026-09-01",
    "check_out": "2026-09-05",
    "nights": 4,
    "available_rooms": [
      {
        "id": "7856de3c-e81c-49e2-83dd-c86996847b70",
        "code": "OVS-01",
        "name": "Ocean View Suite",
        "sensory_match_score": 80,
        "nightly_rate": 450.00,
        "total_price": 1800.00,
        "currency": "USD",
        "bed_type": "King Bed",
        "view_type": "Ocean View",
        "sensory_experience": "Around sunrise you will usually hear the ocean waves before you see them...",
        "booking_url": "https://amanwella.com/reservations?checkin=2026-09-01&checkout=2026-09-05&group_adults=2",
        "booking_platform": "direct",
        "booking_action_label": "Book Direct",
        "destination_source": "property_default",
        "has_booking_destination": true,
        "available_options": [
          {
            "channel": "direct",
            "name": "Official Resort Direct Website",
            "url": "https://amanwella.com/reservations?checkin=2026-09-01&checkout=2026-09-05&group_adults=2",
            "is_preferred": true,
            "source": "property_default",
            "verification_status": "verified"
          },
          {
            "channel": "agoda",
            "name": "Agoda",
            "url": "https://www.agoda.com/amanwella-resort/hotel/tangalle-lk.html?checkin=2026-09-01&checkout=2026-09-05&group_adults=2",
            "is_preferred": false,
            "source": "property_default",
            "verification_status": "verified"
          },
          {
            "channel": "booking_com",
            "name": "Booking.com",
            "url": "https://www.booking.com/hotel/lk/maalu-maalu-resorts.en-gb.html?aid=304142&checkin=2026-09-01&checkout=2026-09-05&group_adults=2",
            "is_preferred": false,
            "source": "legacy_auto",
            "verification_status": "verified"
          }
        ],
        "badges": ["Best Rate Guarantee", "Direct Hotel Booking"]
      }
    ]
  }
}
```

---

## 2. Get Property Booking Destinations & Setup Status
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/booking/destinations`
* **Authentication**: Public — read server-to-server by the guest booking agent, same as the
  other `GET` content endpoints across the platform.

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Booking destinations retrieved successfully",
  "data": {
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "booking_mode": "external",
    "preferred_booking_channel": "direct",
    "destinations": [
      {
        "id": "dest-direct",
        "channel": "direct",
        "name": "Official Resort Direct Website",
        "url": "https://amanwella.com/reservations",
        "is_enabled": true,
        "is_preferred": true,
        "source": "manual",
        "verification_status": "verified",
        "last_verified_at": "2026-08-18T10:25:27Z"
      },
      {
        "id": "dest-agoda",
        "channel": "agoda",
        "name": "Agoda",
        "url": "https://www.agoda.com/amanwella-resort/hotel/tangalle-lk.html",
        "is_enabled": true,
        "is_preferred": false,
        "source": "manual",
        "verification_status": "verified"
      }
    ],
    "rooms": [
      {
        "room_id": "7856de3c-e81c-49e2-83dd-c86996847b70",
        "room_name": "Ocean View Suite",
        "room_code": "OVS-01",
        "has_override": false,
        "destinations": []
      }
    ],
    "setup_status": {
      "booking_mode": "external",
      "active_channel_provider": "direct",
      "preferred_channel": "direct",
      "total_configured_channels": 2,
      "has_direct_website": true,
      "has_booking_com": false,
      "has_agoda": true,
      "has_expedia": false,
      "live_pms_connected": false,
      "aura_managed_reservations": false
    }
  }
}
```

---

## 3. Update Property Booking Destinations
* **HTTP Method**: `PUT`
* **Path**: `/api/v1/properties/{id}/booking/destinations`
* **Authentication**: Required — staff JWT scoped to this property.

### Request Body
```json
{
  "booking_mode": "external",
  "preferred_booking_channel": "direct",
  "destinations": [
    {
      "id": "dest-1",
      "channel": "direct",
      "name": "Official Website",
      "url": "https://yourhotel.com/reservations",
      "is_enabled": true,
      "is_preferred": true,
      "source": "manual",
      "verification_status": "verified"
    },
    {
      "id": "dest-2",
      "channel": "booking_com",
      "name": "Booking.com",
      "url": "https://www.booking.com/hotel/lk/your-hotel.html",
      "is_enabled": true,
      "is_preferred": false,
      "source": "manual",
      "verification_status": "verified"
    }
  ]
}
```

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Booking destinations updated successfully",
  "data": {
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "booking_mode": "external",
    "preferred_booking_channel": "direct",
    "destinations": [...]
  }
}
```

---

## 4. Test Destination Link & Update Verification
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/destinations/test`
* **Authentication**: Required — staff JWT scoped to this property.

### Request Body
```json
{
  "channel": "direct",
  "url": "https://yourhotel.com/reservations"
}
```

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Destination tested and verified",
  "data": {
    "channel": "direct",
    "url": "https://yourhotel.com/reservations",
    "verification_status": "verified",
    "tested_at": "2026-08-18T10:28:34.665Z"
  }
}
```

---

## 5. Update Room-Specific Booking Destination Override
* **HTTP Method**: `PUT`
* **Path**: `/api/v1/properties/{id}/rooms/{roomId}/booking-destinations`
* **Authentication**: Required — staff JWT scoped to this property.

### Request Body
```json
{
  "booking_url": "https://yourhotel.com/book/ocean-view-suite",
  "destinations": [
    {
      "id": "override-1",
      "channel": "direct",
      "name": "Ocean View Suite Direct Landing",
      "url": "https://yourhotel.com/book/ocean-view-suite",
      "is_enabled": true,
      "is_preferred": true,
      "source": "manual",
      "verification_status": "verified"
    }
  ]
}
```

*Note: Send an empty array `destinations: []` and `booking_url: ""` to revert the room back to inheriting property default settings.*

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Room booking destination updated successfully",
  "data": {
    "room_id": "7856de3c-e81c-49e2-83dd-c86996847b70",
    "has_override": true,
    "booking_url": "https://yourhotel.com/book/ocean-view-suite",
    "destinations": [...]
  }
}
```

---

## 6. Track Outbound Booking Click
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/track-click`
* **Authentication**: Public — fired directly from the guest's browser when they click a
  booking CTA.

Asynchronously records when a guest clicks a room card's booking CTA button or selects an option from the multi-channel dropdown popover.

### Request Body
```json
{
  "room_id": "7856de3c-e81c-49e2-83dd-c86996847b70",
  "room_name": "Ocean View Suite",
  "room_code": "OVS-01",
  "channel": "direct",
  "destination_url": "https://amanwella.com/reservations",
  "source": "property_default",
  "session_id": "sess-guest-1049"
}
```

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Booking intent click logged",
  "data": {
    "id": "c1bdbc3a-a8c7-42ac-b6d4-ac0cae5ca86e",
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "room_name": "Ocean View Suite",
    "channel": "direct",
    "created_at": "2026-08-18T10:28:34.665Z"
  }
}
```

---

## 7. Get Outbound Booking Traffic & Intent Analytics
* **HTTP Method**: `GET`
* **Path**: `/api/v1/properties/{id}/booking/analytics`
* **Authentication**: Required — staff JWT scoped to this property.

### Response (`200 OK`)
```json
{
  "status": "success",
  "message": "Booking analytics generated successfully",
  "data": {
    "property_id": "a8360f9a-445f-405a-9725-232113f382b4",
    "total_booking_clicks": 142,
    "total_room_views": 850,
    "overall_ctr": 16.7,
    "top_channel": "Official Hotel Website",
    "channel_breakdown": [
      {
        "channel": "direct",
        "label": "Official Hotel Website",
        "click_count": 94,
        "percentage": 66.2
      },
      {
        "channel": "booking_com",
        "label": "Booking.com",
        "click_count": 32,
        "percentage": 22.5
      },
      {
        "channel": "agoda",
        "label": "Agoda",
        "click_count": 16,
        "percentage": 11.3
      }
    ],
    "room_breakdown": [
      {
        "room_code": "OVS-01",
        "room_name": "Ocean View Suite",
        "click_count": 68,
        "estimated_views": 320,
        "click_through_rate": 21.25,
        "top_channel": "Official Hotel Website"
      },
      {
        "room_code": "BPV-02",
        "room_name": "Beachfront Plunge Pool Villa",
        "click_count": 45,
        "estimated_views": 250,
        "click_through_rate": 18.0,
        "top_channel": "Official Hotel Website"
      }
    ],
    "daily_trend": [
      { "date": "2026-08-17", "clicks": 38 },
      { "date": "2026-08-18", "clicks": 42 }
    ],
    "setup_status": {
      "booking_mode": "external",
      "active_channel_provider": "direct",
      "preferred_channel": "direct",
      "total_configured_channels": 3,
      "has_direct_website": true,
      "has_booking_com": true,
      "has_agoda": true,
      "has_expedia": false,
      "live_pms_connected": false,
      "aura_managed_reservations": false
    }
  }
}
```

---

## 8. Price Breakdown Calculation
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/price-breakdown`
* **Authentication**: Public.

### Request Body
```json
{
  "room_type_slug": "ocean-view-suite",
  "check_in": "2026-09-01",
  "check_out": "2026-09-05",
  "adults": 2
}
```

### Response (`200 OK`)
```json
{
  "status": "success",
  "data": {
    "nights": 4,
    "nightly_rate": 450.00,
    "subtotal": 1800.00,
    "tax_amount": 180.00,
    "service_charge": 180.00,
    "resort_credit": 100.00,
    "total_amount": 2060.00,
    "currency": "USD"
  }
}
```

---

## 9. Create Temporary Booking Hold (15-Min TTL)
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/hold`
* **Authentication**: Public.

Used in Integrated Mode or testing simulator to create a temporary 15-minute room lock.

### Request Body
```json
{
  "guest_name": "Shakthi Warnakulasuriya",
  "guest_email": "shakthi@example.com",
  "guest_phone": "+94771234567",
  "room_type_slug": "ocean-view-suite",
  "check_in": "2026-09-01",
  "check_out": "2026-09-05",
  "adults": 2,
  "children": 0,
  "special_requests": "High floor, quiet corner, early check-in requested."
}
```

### Response (`201 Created`)
```json
{
  "status": "success",
  "data": {
    "hold_id": "hld_9f8e7d6c5b4a",
    "expires_at": "2026-08-18T10:45:00Z",
    "ttl_seconds": 900,
    "status": "held",
    "room_type": "Ocean View Suite",
    "total_amount": 2060.00,
    "currency": "USD"
  }
}
```

---

## 10. Generate Checkout Payment Link
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/payment-link`
* **Authentication**: Public.

### Request Body
```json
{
  "hold_id": "hld_9f8e7d6c5b4a",
  "currency": "USD"
}
```

### Response (`200 OK`)
```json
{
  "status": "success",
  "data": {
    "payment_link_id": "pl_1a2b3c4d5e6f",
    "checkout_url": "https://pay.stripe.com/c/pay/cs_live_sample",
    "amount_due": 2060.00,
    "currency": "USD",
    "expires_at": "2026-08-18T10:45:00Z",
    "status": "active"
  }
}
```
