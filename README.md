# AI Luxury Hospitality Platform Documentation Index

Welcome to the backend REST API & AI Agent Architecture documentation for the **AI Luxury Hospitality Platform** (`chatbot-demo-api` & `chatbot-demo-admin`).

---

## 📚 Documentation Architecture

| Document | Description | Key Components |
| :--- | :--- | :--- |
| 🔑 **[Authentication & Tenant Security API](auth_api.md)** | User authentication, JWT issuance, HttpOnly cookies, tenant registration, and cross-tenant boundary security. | `/api/v1/auth/login`, `/register`, `/me`, `/logout` |
| 🏰 **[Property Management API](property_management_api.md)** | Core property profile, room categories, dining outlets, media assets, heritage stories, attractions, and bot config. | `/api/v1/properties`, `/rooms`, `/outlets`, `/media`, `/bot-config` |
| 🌐 **[Omnichannel Booking Destinations & Distribution](booking_destinations_and_channels.md)** | Multi-channel hotel distribution architecture, 4-tier precedence engine, query parameter injection, room overrides, and outbound click analytics. | 4-Tier Precedence Engine, Direct, Booking.com, Agoda, Expedia |
| 🏨 **[Conversational Booking Engine API](booking_engine_api.md)** | Room availability checking, destination resolution, itemized price breakdowns, 15-minute booking holds, payment link generation, and outbound click tracking. | `/booking/check-availability`, `/booking/destinations`, `/booking/track-click`, `/booking/analytics` |
| 🔄 **[Room Reservation & Booking Flow](booking_and_reservation_flow.md)** | Full architectural analysis of the room reservation lifecycle with sequence diagrams, component classifications (Real vs. Simulated vs. Stubs), and V1 external booking flow. | Complete Architecture & Audit (v3.0) |
| 🗺️ **[Experience Timeline & Itinerary API](itinerary_experience_api.md)** | Multi-tenant experience layer, in-hotel activities, partner programs, local attractions, and multi-day itinerary generation. | `/api/v1/properties/{id}/experiences`, `/itinerary/generate`, `/experience-timeline` |
| 💬 **[Chat Engine & SSE Gateway API](chat_gateway_api.md)** | Multi-turn Server-Sent Events (SSE) streaming, subagent routing, generative UI payload delivery, and chat history. | `/chat/completions`, `/chat/sessions/{id}/history`, `/chat/feedback` |
| 🧠 **[Knowledge Engine (RAG) API](knowledge_engine_api.md)** | Vector ingestion (Qdrant Cloud), PDF/MD knowledge chunking, semantic hybrid search, and property synchronization. | `/knowledge/ingest`, `/knowledge/sync`, `/knowledge/search` |
| 👤 **[Session & Guest Memory API](memory_controller.md)** | Short-term session preferences, long-term guest profiling, background memory extraction, and session-to-guest binding. | `/memory/session`, `/memory/bind` |
| 📋 **[Project Brief & System Design](project_brief.md)** | Platform specifications, multi-agent architecture, guardrails, and PMS adapter integration requirements. | Architecture Specs & Persona Guidelines |
| ✨ **[Platform Feature Compendium](platform_features.md)** | Full audit of every feature across frontend + backend with storytelling context, complementary relationships, and the connected guest journey. | All Features & Omnichannel Booking |

---

## ⚡ Server Execution & Quick Start

### 1. Go API Backend (`chatbot-demo-api`)
```bash
cd chatbot-demo-api
go run main.go
# Server starts on http://localhost:8080
```

### 2. Next.js Admin & Frontend UI (`chatbot-demo-admin`)
```bash
cd chatbot-demo-admin
npm run dev
# Server starts on http://localhost:3000
```
