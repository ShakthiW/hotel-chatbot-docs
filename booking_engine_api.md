# Conversational Booking & Availability Engine API Specification

The **Conversational Booking & Availability Engine API** provides room inventory checking, price breakdown calculation, 15-minute temporary booking holds, and secure payment checkout link generation for the AI Hospitality Platform.

---

## Endpoints Overview

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/v1/properties/{id}/booking/check-availability` | Checks live room category availability & nightly rates. |
| `POST` | `/api/v1/properties/{id}/booking/price-breakdown` | Calculates itemized taxes, service charges, and resort credits. |
| `POST` | `/api/v1/properties/{id}/booking/hold` | Creates a 15-minute temporary room hold reservation. |
| `POST` | `/api/v1/properties/{id}/booking/payment-link` | Generates a payment link and checkout UI payload. |

---

## 1. Check Room Availability
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/check-availability`

### Request Body
```json
{
  "check_in": "2026-12-10",
  "check_out": "2026-12-15",
  "adults": 2,
  "children": 1,
  "room_type_slug": "ocean-view-suite"
}
```

### Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "available": true,
    "room_types": [
      {
        "id": "e7b1a2c3-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
        "name": "Ocean View Suite",
        "slug": "ocean-view-suite",
        "base_price": 450.00,
        "currency": "USD",
        "max_occupancy_adults": 2,
        "max_occupancy_children": 1,
        "view_type": "Ocean View",
        "bed_type": "King Bed",
        "size_sqm": 75.5,
        "available_rooms_count": 3
      }
    ]
  }
}
```

---

## 2. Price Breakdown Calculation
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/price-breakdown`

### Request Body
```json
{
  "room_type_slug": "ocean-view-suite",
  "check_in": "2026-12-10",
  "check_out": "2026-12-15",
  "adults": 2
}
```

### Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "nights": 5,
    "nightly_rate": 450.00,
    "subtotal": 2250.00,
    "tax_amount": 225.00,
    "service_charge": 225.00,
    "resort_credit": 100.00,
    "total_amount": 2600.00,
    "currency": "USD"
  }
}
```

---

## 3. Create Temporary Booking Hold (15-Min TTL)
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/hold`

### Request Body
```json
{
  "guest_name": "Shakthi Warnakulasuriya",
  "guest_email": "shakthi@example.com",
  "guest_phone": "+94771234567",
  "room_type_slug": "ocean-view-suite",
  "check_in": "2026-12-10",
  "check_out": "2026-12-15",
  "adults": 2
}
```

### Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "hold_id": "HOLD-98214-OVS",
    "status": "held",
    "expires_at": "2026-12-10T14:45:00Z",
    "ttl_seconds": 900,
    "room_name": "Ocean View Suite",
    "total_amount": 2600.00,
    "currency": "USD"
  }
}
```

---

## 4. Generate Payment Checkout Link
* **HTTP Method**: `POST`
* **Path**: `/api/v1/properties/{id}/booking/payment-link`

### Request Body
```json
{
  "hold_id": "HOLD-98214-OVS"
}
```

### Response (`200 OK`)
```json
{
  "success": true,
  "data": {
    "hold_id": "HOLD-98214-OVS",
    "payment_url": "http://localhost:3000/checkout?holdId=HOLD-98214-OVS",
    "status": "ready",
    "amount": 2600.00,
    "currency": "USD"
  }
}
```
