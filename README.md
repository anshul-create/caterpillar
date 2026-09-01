# Smart Rental Tracking System

A smart asset rental tracking platform for construction/mining equipment — real-time tracking, check-in/check-out, usage logging, overdue alerts, demand forecasting, and anomaly detection.

Built for [Hackathon Name] by [Team Name].

---

## Problem Statement

Companies renting machinery (excavators, cranes, bulldozers, graders) through registered dealers still manage rentals manually or via spreadsheets. This causes:
- Equipment being lost or unaccounted for
- Delays and downtime due to misallocation
- Unexpected rental extensions and costs

This project builds a system to track equipment in real time, predict demand, flag under-utilized/misused assets, and log usage automatically.

---

## Features (Expected Outcomes)

- [ ] **Asset Dashboard** — live status list of all rented equipment
- [ ] **Check-in / Check-out System** — QR code / RFID simulated or manual entry
- [ ] **Usage Logging** — runtime hours, fuel usage, location, idle hours; summarized per site
- [ ] **Overdue Alerts & Notifications** — remind users as return date approaches
- [ ] **Demand Forecasting** — predict which equipment will be needed at which site/time
- [ ] **Anomaly Detection** — flag long idle hours, unassigned equipment, misuse patterns

---

## Tech Stack

| Layer | Choice |
|---|---|
| Frontend | React (Vite) + Tailwind CSS + shadcn/ui |
| Charts | Recharts |
| Backend | FastAPI (Python) |
| Database | PostgreSQL (Supabase/Railway for hosting) |
| Forecasting | pandas + scikit-learn (rolling average / linear regression) |
| Anomaly Detection | Rule-based thresholds (+ optional Isolation Forest) |
| QR Simulation | `qrcode.react` on frontend |
| Deployment | Vercel (frontend), Railway/Render (backend + DB) |

---

## Data Model (from problem statement sample data)

**Equipment**
| Field | Type |
|---|---|
| equipment_id | string (PK) |
| type | string (Excavator, Crane, Bulldozer, Grader) |
| site_id | string (FK, nullable) |

**RentalLog**
| Field | Type |
|---|---|
| log_id | int (PK) |
| equipment_id | FK |
| site_id | FK (nullable) |
| check_out_date | date |
| check_in_date | date |
| engine_hours_per_day | float |
| idle_hours_per_day | float |
| operating_days | int |
| last_operator_id | string (nullable) |

**Site**
| Field | Type |
|---|---|
| site_id | string (PK) |
| name | string |
| location | string |

**Alert**
| Field | Type |
|---|---|
| alert_id | int (PK) |
| equipment_id | FK |
| type | enum (overdue, idle, unassigned, anomaly) |
| message | string |
| created_at | timestamp |

---

## How to Approach This Project (Hackathon Plan)

### Phase 1 — Setup & Data (first 2–3 hours)
1. Set up repo structure (`/frontend`, `/backend`).
2. Design and create the Postgres schema (Equipment, Site, RentalLog, Alert, Operator).
3. Seed the database using the sample data given in the problem statement image — this guarantees your demo has realistic data from minute one.
4. Get FastAPI + Postgres talking (basic `GET /equipment` returning seed data) and confirm the frontend can fetch it. Getting this thin vertical slice working early avoids integration panic later.

### Phase 2 — Core CRUD Features (next 3–4 hours)
5. Build the **Asset Dashboard**: table/cards showing equipment, type, site, status (rented/available/overdue), computed from check-out/check-in dates.
6. Build **Check-in/Check-out**: a simple form or simulated QR scan page (`/checkin/:equipment_id`) that writes a new log row and updates equipment status.
7. Build **Usage Logging** view: per-equipment and per-site summaries — total rented hours, idle hours, downtime — using simple aggregation queries.

### Phase 3 — Smart Features (next 3–4 hours)
8. **Overdue Alerts**: a scheduled/on-demand check comparing `check_in_date` (expected return) vs current date; flag equipment past due, surface on dashboard.
9. **Anomaly Detection (rule-based first)**: flag equipment where idle_hours/day > threshold, or `last_operator_id` is NULL (unassigned), or engine_hours is 0 for extended periods. This alone satisfies the requirement — don't over-invest in ML unless time allows.
10. **Demand Forecasting**: aggregate historical check-outs by site + equipment type over time; use a simple rolling average or linear regression to predict next period's likely demand. Keep it explainable — a chart showing "predicted vs historical" is more convincing to judges than a black-box model.

### Phase 4 — Polish & Demo Prep (last 1–2 hours)
11. Add loading states, empty states, and make the dashboard visually clean (this matters a lot for hackathon judging).
12. Prepare a 2–3 minute demo script: show dashboard → simulate a check-out/check-in → show an overdue alert firing → show an anomaly flagged → show a forecast chart.
13. Write this README fully (architecture diagram, setup instructions, screenshots) — judges often skim it before the demo.

### Suggested Team Split (if 3–4 people)
- **1 person:** Backend API + DB schema + seed data
- **1 person:** Frontend dashboard + check-in/out UI
- **1 person:** Forecasting + anomaly detection logic (can be pure Python/pandas scripts wired into an endpoint)
- **1 person:** Alerts/notifications + polish + demo prep + README/pitch

### Practical tips
- Don't build real RFID/hardware integration — simulate it. Judges care about the concept working end-to-end, not physical hardware.
- Rule-based anomaly detection is a legitimate, defensible MVP — only add ML if there's spare time.
- Get one full slice (DB → API → UI) working before dividing into parallel workstreams — it de-risks integration issues later.
- Keep the forecasting model simple and interpretable; you'll need to explain it live to judges.

---

## Local Setup

### Backend
```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

### Environment Variables
```
DATABASE_URL=postgresql://user:password@host:port/dbname
```

---

## API Endpoints (planned)

| Method | Endpoint | Description |
|---|---|---|
| GET | `/equipment` | List all equipment with live status |
| POST | `/checkin/{equipment_id}` | Check equipment back in |
| POST | `/checkout/{equipment_id}` | Check equipment out |
| GET | `/usage-logs` | Usage summary (per site/equipment) |
| GET | `/alerts` | Active alerts (overdue, anomalies) |
| GET | `/forecast` | Demand forecast per site/equipment type |

---

## Team

| Name | Role |
|---|---|
| | |
| | |
| | |

## License

MIT (or update as needed for your hackathon submission)