# 🚚 LogiTrack Pro — Last-Mile Delivery Management Platform

A backend-driven delivery management platform where customers and admins create orders with **auto-calculated charges**, orders are routed to the **nearest available agent**, and every status change is logged in an **immutable tracking timeline** with customer notifications at each step.

Built with **FastAPI + SQLAlchemy + SQLite** (Postgres-ready), a lightweight HTML/JS frontend, JWT-based **role auth** (Customer / Agent / Admin), and a fully **admin-configurable** rate engine — no hardcoded prices or zones.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Project Structure](#project-structure)
3. [Setup Guide](#setup-guide)
4. [Environment Variables](#environment-variables-envexample)
5. [Seeded Demo Accounts](#seeded-demo-accounts)
6. [Database Schema](#database-schema)
7. [Rate Calculation Engine](#rate-calculation-engine)
8. [Auto-Assignment Engine](#auto-assignment-engine)
9. [Order Lifecycle & Failed Delivery Flow](#order-lifecycle--failed-delivery-flow)
10. [API Documentation](#api-documentation)
11. [Running Tests](#running-tests)
12. [Docker](#docker)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend API | FastAPI (Python 3.9+) |
| ORM / DB | SQLAlchemy 2.0, SQLite (default) — swappable to PostgreSQL |
| Auth | JWT (python-jose) + passlib/bcrypt password hashing, role-based access control |
| Frontend | Static HTML / CSS / vanilla JS (served by FastAPI at `/`) |
| Testing | Pytest (`tests/`) |
| Containerization | Docker / docker-compose |

---

## Project Structure

```
unthinkable/
├── app/
│   ├── main.py               # FastAPI app entrypoint, router mounting, static frontend
│   ├── config.py             # Settings (env-driven)
│   ├── database.py           # SQLAlchemy engine/session
│   ├── models.py             # ORM models (User, Zone, RateCard, Order, OrderTimeline, ...)
│   ├── schemas.py            # Pydantic request/response schemas
│   ├── seed.py                # Auto-seeds demo users, zones, rate cards, agents on startup
│   ├── routers/
│   │   ├── auth.py           # /api/auth  — register, login, me
│   │   ├── zones.py          # /api/zones — CRUD (admin)
│   │   ├── rates.py          # /api/rates — rate cards CRUD + price preview calculator
│   │   ├── agents.py         # /api/agents — agent profile & status management
│   │   ├── orders.py         # /api/orders — create, list, track, assign, status, reschedule
│   │   └── notifications.py  # /api/notifications — outbox / notification log
│   └── services/
│       ├── auth_service.py       # password hashing, JWT issue/verify, role guards
│       ├── rate_engine.py        # zone detection + volumetric weight + charge calculation
│       ├── assignment_engine.py  # Haversine distance + multi-factor agent scoring
│       ├── lifecycle_service.py  # order status transitions + immutable timeline writes
│       └── notification_service.py # notification outbox (email/SMS simulation)
├── static/                   # index.html / app.js / style.css (frontend)
├── tests/                    # pytest suite (API, assignment, lifecycle, rate engine)
├── docs/system_design.md     # System design write-up
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── .env.example
```

---

## Setup Guide

### Prerequisites
- Python 3.9+
- pip

### 1. Clone & create a virtual environment

```bash
git clone <your-repo-url>
cd unthinkable
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure environment

```bash
cp .env.example .env
# edit .env if you want to change the secret key, DB URL, or enable SMTP/Twilio
```

### 4. Run the app

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

On first boot, the app **auto-creates tables** and **auto-seeds** demo data (users, zones, rate cards, agents, a couple of sample orders) — see [Seeded Demo Accounts](#seeded-demo-accounts).

- Frontend: http://localhost:8000/
- Swagger / interactive API docs: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc
- Health check: http://localhost:8000/api/health

### 5. (Optional) Reset the database

```bash
rm delivery_tracker.db
uvicorn app.main:app --reload   # re-seeds automatically on next start
```

---

## Environment Variables (`.env.example`)

```env
# Server
PORT=8000
HOST=0.0.0.0
DEBUG=True

# Security
SECRET_KEY=super-secret-jwt-key-for-last-mile-tracker-2026
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# Database Configuration (SQLite default; PostgreSQL supported)
DATABASE_URL=sqlite:///./delivery_tracker.db
# DATABASE_URL=postgresql://user:password@localhost:5432/logitrack_db

# Notification Service (optional third-party gateways; in-app outbox active by default)
EMAIL_SERVICE_ENABLED=False
SMS_SERVICE_ENABLED=False
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
SMTP_USER=apikey
SMTP_PASSWORD=your-sendgrid-api-key
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-token
TWILIO_PHONE_NUMBER=+1234567890
```

> Notifications work out of the box without SMTP/Twilio: every notification is written to the `notification_logs` table as an **outbox** and visible via `/api/notifications`. Setting the `*_SERVICE_ENABLED` flags and credentials lets you wire in a real provider without touching the calling code.

---

## Seeded Demo Accounts

| Role | Email | Password |
|---|---|---|
| Admin | `admin@logitrack.com` | `admin123` |
| Customer (B2B) | `john.doe@apexindustries.com` | `customer123` |
| Customer (B2C) | `sarah.jenkins@gmail.com` | `customer123` |
| Agent | `rajesh.kumar@logitrack.com` | `agent123` |
| Agent | `vikram.singh@logitrack.com` | `agent123` |
| Agent | `amit.sharma@logitrack.com` | `agent123` |

Seed data also includes multiple **zones** (with pincode/area mappings), **B2B/B2C rate cards** (intra-zone & inter-zone), and agent profiles with vehicle types and live coordinates — enough to exercise rate calculation and auto-assignment immediately.

---

## Database Schema

| Table | Purpose | Key Fields |
|---|---|---|
| **users** | All accounts (customer / agent / admin) | `role`, `hashed_password` |
| **zones** | Delivery zones | `pincodes[]`, `areas[]`, `center_lat/lng`, `radius_km` |
| **rate_cards** | Admin-configurable pricing rules | `zone_type` (INTRA/INTER/SPECIFIC_PAIR), `order_type` (B2B/B2C), `base_weight_kg`, `base_rate`, `per_kg_rate`, `cod_fixed_surcharge`, `cod_percentage_surcharge` |
| **agent_profiles** | Delivery agent operational state | `vehicle_type`, `max_capacity`, `current_load`, `status`, `current_lat/lng`, `assigned_zone_id` |
| **orders** | Core order record | pickup/drop address + zone + coords, dimensions, `volumetric_weight_kg`, `chargeable_weight_kg`, charges breakdown, `status`, `assigned_agent_id`, reschedule fields |
| **order_timelines** | **Immutable, append-only** audit trail | `from_status → to_status`, `actor_id/role/name`, `timestamp`, `location`, `notes`, `failure_reason` |
| **notification_logs** | Notification outbox | `channel` (EMAIL/SMS), `event_type`, `title`, `body`, `status` |

Relationships: a `User` can be a customer on many `Order`s and (if role=AGENT) has one `AgentProfile`; each `Order` belongs to a pickup `Zone` and a drop `Zone`; each `Order` has many ordered `OrderTimeline` entries and `NotificationLog` entries.

---

## Rate Calculation Engine

Implemented in `app/services/rate_engine.py` — **fully dynamic, zero hardcoded prices**.

**Pipeline (`RateEngine.calculate_rate`):**

1. **Zone detection** (`detect_zone`) for both pickup and drop, in priority order:
   1. Explicit `zone_id` if the client passed one
   2. Exact **pincode** match against `Zone.pincodes`
   3. **Area/locality keyword** match inside the address or city against `Zone.areas`
   4. **City name** match
   5. Fallback to the first active zone
2. **Volumetric weight**: `(L × W × H) / 5000` (cm → kg)
3. **Chargeable weight**: `max(actual_weight_kg, volumetric_weight_kg)` — the response reports whether billing used `"actual"` or `"volumetric"`.
4. **Rate card lookup** (`get_matching_rate_card`), in priority order:
   1. A `SPECIFIC_PAIR` rate card for this exact (pickup zone, drop zone, order type)
   2. Otherwise `INTRA_ZONE` (same pickup/drop zone) or `INTER_ZONE` (different zones) card for the order's `B2B`/`B2C` type
   3. Fallback to any active card for that order type
   4. If none exists → `404`, forcing admins to configure rate cards rather than silently charging ₹0
5. **Charge breakdown**:
   - `base_charge` = card's `base_rate` (covers up to `base_weight_kg`)
   - `weight_charge` = `max(0, chargeable_weight − base_weight_kg) × per_kg_rate`
   - `cod_surcharge` (only if `payment_type == COD`) = `cod_fixed_surcharge + (order_value × cod_percentage_surcharge / 100)`
   - `total_charge` = `base_charge + weight_charge + cod_surcharge`

This is exposed standalone as a **pre-confirmation preview** via `POST /api/rates/calculate` (used by the frontend's live calculator) and is re-run server-side when an order is actually created, so the quoted price and the billed price always agree.

Admins manage the underlying rules via `GET/POST/PUT/DELETE /api/rates/cards` and `/api/zones` — separate cards per zone-pair type and per B2B/B2C, so pricing can be tuned with no code changes.

---

## Auto-Assignment Engine

Implemented in `app/services/assignment_engine.py`.

1. **Availability filter**: only agents with `status == ON_DUTY` and `current_load < max_capacity` are considered. `current_load` is recalculated live by counting each agent's orders in a non-terminal state (`ASSIGNED`, `PICKED_UP`, `IN_TRANSIT`, `OUT_FOR_DELIVERY`).
2. **Proximity**: great-circle distance from each candidate agent's live `(current_lat, current_lng)` to the order's pickup coordinates (or pickup-zone center as fallback) via the **Haversine formula**.
3. **Multi-factor composite score** (lower = better):
   - `distance_km × 1.5`
   - `+ workload_penalty` = `(current_load / max_capacity) × 15`
   - `+ zone_affinity_penalty` = `0` if the agent's assigned zone matches the pickup zone, else `10`
   - `+ vehicle_penalty` = `50` if the parcel is heavy (`> 20kg`) or **B2B** but the candidate rides a `BIKE` (prefers Van/Truck for bulk/heavy freight)
4. The agent with the **lowest total score** is selected and assigned; `current_load` is recalculated afterward.

Exposed via `POST /api/orders/{order_id}/assign` — admins can either pass an explicit `agent_id` for a manual override, or omit it to trigger auto-assignment. All manual assignments are attributable through the actor fields on the resulting timeline entry.

---

## Order Lifecycle & Failed Delivery Flow

**State machine** (`app/services/lifecycle_service.py`):

```
CREATED → ASSIGNED → PICKED_UP → IN_TRANSIT → OUT_FOR_DELIVERY → DELIVERED
                                                       └────────→ FAILED → RESCHEDULED → (re-assigned, re-enters ASSIGNED)
```
`CANCELLED` is reachable as an admin override from most pre-delivery states.

- Every transition — whether made by an agent, customer, or admin override — writes a new **`OrderTimeline`** row capturing `from_status`, `to_status`, `actor_id/role/name`, `timestamp`, optional `location`, `notes`, and `failure_reason`. Rows are **append-only**; nothing is edited or deleted, so the full history is always reconstructible via `GET /api/orders/{id}` (embeds ordered `timeline_entries`) or `GET /api/orders/track/{tracking_number}` for customer self-tracking.
- On **every status change**, a notification is written to the outbox (`NotificationLog`) and, in this build, "sent" via email — visible under `GET /api/notifications`.
- **Failed delivery flow**:
  1. Agent (or admin) sets status to `FAILED` with a `failure_reason` → timeline entry logged, customer notified.
  2. Customer (or admin on their behalf) calls `POST /api/orders/{id}/reschedule` with a new date/notes → order moves to `RESCHEDULED`, `rescheduled_date`/`reschedule_notes` stored.
  3. The order re-enters the assignment pool: `POST /api/orders/{id}/assign` (auto or manual) picks an agent for the new attempt, moving status back to `ASSIGNED`.
- Admins can **override any order's status directly** (`PUT /api/orders/{id}/status`) regardless of the "natural" actor, for exception handling — still fully logged in the timeline.

---

## API Documentation

Full interactive documentation (request/response schemas, try-it-out) is auto-generated by FastAPI:

- **Swagger UI** → `/docs`
- **ReDoc** → `/redoc`

### Endpoint summary

All routes are prefixed with `/api`.

| Router | Method & Path | Description | Access |
|---|---|---|---|
| Auth | `POST /auth/register` | Register a new user (customer/agent/admin) | Public |
| | `POST /auth/login` | Login, returns JWT | Public |
| | `GET /auth/me` | Current user profile | Authenticated |
| Zones | `GET /zones/` | List zones | Any authenticated |
| | `POST /zones/` | Create zone | Admin |
| | `GET/PUT/DELETE /zones/{id}` | Get/update/delete a zone | Admin (write) |
| Rates | `POST /rates/calculate` | **Pre-confirmation price preview** | Any authenticated |
| | `GET /rates/cards` | List rate cards | Any authenticated |
| | `POST/PUT/DELETE /rates/cards[/{id}]` | Manage rate cards | Admin |
| Agents | `GET /agents/` | List agent profiles | Admin |
| | `GET /agents/me` | Own agent profile | Agent |
| | `PUT /agents/me/status` | Update own availability/location | Agent |
| | `PUT /agents/{id}` | Admin edits any agent | Admin |
| Orders | `POST /orders/` | Create order (auto rate calc + zone detect) | Customer / Admin (on behalf) |
| | `GET /orders/` | List orders (filter by status/zone/agent) | Role-scoped |
| | `GET /orders/track/{tracking_number}` | Public tracking lookup | Public/Customer |
| | `GET /orders/{id}` | Order detail incl. full timeline | Role-scoped |
| | `PUT /orders/{id}/status` | Update/override status | Agent / Admin |
| | `POST /orders/{id}/assign` | Manual or auto agent assignment | Admin |
| | `POST /orders/{id}/reschedule` | Reschedule a failed delivery | Customer / Admin |
| Notifications | `GET /notifications/` | List notification outbox | Admin |
| | `GET /notifications/order/{id}` | Notifications for one order | Role-scoped |

Auth: send `Authorization: Bearer <token>` from `/auth/login` on all protected routes.

---

## Running Tests

```bash
pip install -r requirements.txt
pytest -v
```

Test suite (`tests/`) covers the full API surface, the rate engine (volumetric weight, zone-based rate lookup, COD surcharge math), the assignment engine (Haversine scoring, capacity limits), and the full order lifecycle including the failed-delivery/reschedule path.

---

## Docker

```bash
docker-compose up --build
```

Runs migrations/seed then serves the app at `http://localhost:8000`. Set `DATABASE_URL` in `docker-compose.yml` (or `.env`) to point at PostgreSQL for a production deployment.
