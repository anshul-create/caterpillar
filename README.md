# Smart Rental Tracking System

## Background

In industries like construction and mining, companies often rent machinery and tools instead of owning them. However, managing these rentals — tracking where equipment is, who's using it, when it's due to return, and predicting upcoming demand — is still largely manual or spreadsheet-based. This results in:

- Equipment being lost or unaccounted for
- Delays and downtime due to misallocation
- Unexpected rental extensions and costs

## Challenge

Design a smart asset rental tracking system that can:

- Track and monitor rented equipment in real time
- Predict demand, flag under-utilized assets, and optimize rentals
- Log usage and conditions

## Expected Outcomes

| Feature | Description |
|---|---|
| **Asset Dashboard** | List of all rented equipment with live status (data assumed real-time) |
| **Check-in/Check-out System** | Based on QR code/RFID simulation or user entry |
| **Usage Logging** | Runtime hours, fuel usage, location, idle hours. Summary of total rented hours, usage per site, downtime |
| **Overdue Alerts & Notifications** | Remind users when return time is approaching |
| **Demand Forecasting** | Help companies pre-position equipment by predicting which tools/machines will be needed at certain sites/times |
| **Anomaly Detection** | Use historical data to detect misuse of assets, e.g. long idle hours, unassigned equipment |

---

## Tech Stack (MVP)

| Layer | Choice |
|---|---|
| Frontend | React + Tailwind CSS + Recharts |
| Backend | FastAPI (Python) |
| Database | PostgreSQL |
| Forecasting | scikit-learn (Linear Regression) |
| Anomaly Detection | Rule-based (Python + pandas) |
| Check-in/out | `react-qr-code` (generate) + `html5-qrcode` (scan), simulated |
| Alerts | Cron job (APScheduler) + email/notification |
| Hosting | Vercel (frontend) + Render/Railway (backend + DB) |

FastAPI was chosen over Node/Express so the same language (Python) powers both the API and the forecasting/anomaly-detection logic, avoiding extra glue code between services.

---

## Data Model

Core tables derived from the sample dataset:

**equipment**
- `equipment_id`, `type`

**sites**
- `site_id`

**rentals**
- `equipment_id`, `site_id`, `check_out_date`, `check_in_date`

**usage_logs**
- `equipment_id`, `engine_hours_day`, `idle_hours_day`, `operating_days`, `last_operator_id`

**anomalies**
```sql
CREATE TABLE anomalies (
    id SERIAL PRIMARY KEY,
    equipment_id VARCHAR NOT NULL,
    rule_code VARCHAR NOT NULL,       -- e.g. 'UNASSIGNED_SITE'
    severity VARCHAR NOT NULL,        -- 'high' | 'medium' | 'low'
    reason TEXT NOT NULL,             -- human-readable explanation
    detected_at TIMESTAMP DEFAULT NOW(),
    resolved BOOLEAN DEFAULT FALSE
);
```

---

## Anomaly Detection (Rule-Based)

MVP uses transparent, explainable rule-based checks rather than an unsupervised ML model, since the dataset is small and interpretability matters for a first release.

| # | Rule | Condition | Severity |
|---|---|---|---|
| 1 | Unassigned equipment | `site_id IS NULL` while checked out | High |
| 2 | No operator logged | `last_operator_id IS NULL` while checked out | High |
| 3 | Ghost / zero usage | `engine_hours_day = 0` for `operating_days > 3` | Medium |
| 4 | Excessive idle | `idle_hours_day > 8` | Medium |
| 5 | Idle-to-engine ratio | `idle / (idle + engine) > 0.85` | Medium |
| 6 | Overdue return | `check_in_date` missing past expected due date | High |
| 7 | Abnormally long rental | `operating_days > 1.5x avg` for equipment type | Low |
| 8 | Low utilization, long rental | `engine_hours_day < 1` and `operating_days > 10` | Low |

Example from sample data: `EQX1002` and `EQX1007` both have `NULL` site, `NULL` operator, and 0 engine hours — flagged by rules 1–3.

```python
def detect_anomalies(df: pd.DataFrame) -> list[dict]:
    """
    df columns: equipment_id, type, site_id, check_out_date, check_in_date,
    engine_hours_day, idle_hours_day, operating_days, last_operator_id
    """
    anomalies = []
    avg_days_by_type = df.groupby("type")["operating_days"].transform("mean")

    for i, row in df.iterrows():
        checked_out = pd.isna(row["check_in_date"])

        if checked_out and pd.isna(row["site_id"]):
            anomalies.append(_flag(row, "UNASSIGNED_SITE", "high",
                f"{row['equipment_id']} checked out with no site assigned"))

        if checked_out and pd.isna(row["last_operator_id"]):
            anomalies.append(_flag(row, "NO_OPERATOR", "high",
                f"{row['equipment_id']} has no operator logged"))

        if row["engine_hours_day"] == 0 and row["operating_days"] > 3:
            anomalies.append(_flag(row, "ZERO_USAGE", "medium",
                f"{row['equipment_id']} shows 0 engine hours/day over {row['operating_days']} days"))

        if row["idle_hours_day"] > 8:
            anomalies.append(_flag(row, "HIGH_IDLE", "medium",
                f"{row['equipment_id']} idle {row['idle_hours_day']} hrs/day"))

        total = row["engine_hours_day"] + row["idle_hours_day"]
        if total > 0 and (row["idle_hours_day"] / total) > 0.85:
            anomalies.append(_flag(row, "HIGH_IDLE_RATIO", "medium",
                f"{row['equipment_id']} idle ratio {(row['idle_hours_day']/total):.0%}"))

        if row["operating_days"] > avg_days_by_type[i] * 1.5:
            anomalies.append(_flag(row, "LONG_RENTAL", "low",
                f"{row['equipment_id']} rented {row['operating_days']} days, "
                f"vs avg {avg_days_by_type[i]:.0f} for {row['type']}"))

        if row["engine_hours_day"] < 1 and row["operating_days"] > 10:
            anomalies.append(_flag(row, "LOW_UTILIZATION", "low",
                f"{row['equipment_id']} under-utilized: "
                f"{row['engine_hours_day']} hrs/day over {row['operating_days']} days"))

    return anomalies
```

Runs as a scheduled job (daily) or on-demand via `POST /anomalies/scan`. Results are stored in the `anomalies` table and surfaced on the dashboard's Anomalies panel, sorted by severity, each with a human-readable reason.

**Roadmap:** replace/augment with an Isolation Forest model once sufficient historical data accumulates for unsupervised outlier detection.

---

## Demand Forecasting (Linear Regression)

Predicts which equipment types will be needed at which sites in upcoming periods.

**Approach:**
- Aggregate historical rentals by `site_id` + `equipment_type` + time period (week/month)
- Features: past rental frequency, month/season, site, equipment type (one-hot encoded), average operating days
- Target: expected rental count (or rental-days) for the next period
- Model: `scikit-learn.LinearRegression` (or `Ridge` for small datasets, to reduce overfitting)

```python
from sklearn.linear_model import LinearRegression
import pandas as pd

# df: historical rentals aggregated by site + type + month
X = pd.get_dummies(df[["site_id", "equipment_type", "month_num", "past_rentals_count"]])
y = df["target_next_month_rentals"]

model = LinearRegression()
model.fit(X, y)

predictions = model.predict(X_next_period)
```

Chosen for interpretability (coefficients directly explain demand drivers per site/type), fast training, and no tuning overhead — appropriate for an MVP with limited historical data.

**Note:** the sample dataset (7 rows) is far too small to fit a meaningful regression. For demo purposes, synthesize 6–12 months of historical rental records per site, or present this as the model architecture with the caveat that accuracy improves as real historical data accumulates.

---

## API Endpoints (MVP scope)

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/equipment` | Asset dashboard — live status list |
| POST | `/rentals/check-out` | Check out equipment (QR/RFID simulated) |
| POST | `/rentals/check-in` | Check in equipment |
| GET | `/usage-logs/:equipment_id` | Runtime hours, idle hours, fuel, location |
| GET | `/alerts/overdue` | Equipment approaching/past due date |
| GET | `/forecast` | Predicted demand per site/equipment type |
| POST | `/anomalies/scan` | Trigger rule-based anomaly scan |
| GET | `/anomalies` | List flagged anomalies (filter by severity/resolved) |

---

## Actionable Build Steps

1. **Set up repo & environments**
   - Init monorepo (or `/frontend` + `/backend` folders)
   - Set up PostgreSQL instance (local or Render/Railway)
   - Configure `.env` for DB connection string

2. **Build the database schema**
   - Create `equipment`, `sites`, `rentals`, `usage_logs`, `anomalies` tables
   - Seed with the sample dataset (7 equipment rows) for early testing

3. **Build core backend (FastAPI)**
   - `GET /equipment` — asset dashboard endpoint
   - `POST /rentals/check-out` and `POST /rentals/check-in`
   - `GET /usage-logs/:equipment_id`
   - Connect endpoints to PostgreSQL via SQLAlchemy

4. **Build check-in/out simulation**
   - Add QR code generation per equipment (`react-qr-code`)
   - Add scan simulation on frontend (`html5-qrcode` or a manual "scan" button)
   - Wire scan action to check-in/check-out endpoints

5. **Build the Asset Dashboard (frontend)**
   - React + Tailwind list/table view of all equipment with live status
   - Add filters (by site, type, status)
   - Add usage charts (Recharts) for engine hours / idle hours per equipment

6. **Implement overdue alerts**
   - Add `expected_return_date` field to `rentals`
   - Cron job (APScheduler) to check overdue equipment daily
   - Trigger email (SendGrid/Nodemailer) or in-app notification

7. **Implement rule-based anomaly detection**
   - Build `detect_anomalies()` function (rules 1–8 from above)
   - Add `POST /anomalies/scan` endpoint to run it on demand
   - Add `GET /anomalies` endpoint, sorted by severity
   - Build Anomalies panel on dashboard showing equipment + reason

8. **Implement demand forecasting**
   - Generate/synthesize historical rental data (6–12 months) if sample data is insufficient
   - Aggregate by site + equipment type + month
   - Train `LinearRegression` model on aggregated data
   - Add `GET /forecast` endpoint returning predicted demand per site/type
   - Display forecast on dashboard (simple bar/line chart)

9. **Polish & connect end-to-end**
   - Confirm dashboard reflects live check-in/out, usage logs, alerts, anomalies, and forecast in one place
   - Add loading/error states on frontend
   - Smoke-test full flow: check-out → usage logging → overdue alert → anomaly flag → forecast update

10. **Deploy**
    - Frontend → Vercel
    - Backend + DB → Render or Railway
    - Verify environment variables and CORS settings in production

11. **Prepare demo/pitch**
    - Highlight the two anomalies already visible in sample data (EQX1002, EQX1007)
    - Frame rule-based detection and linear regression as MVP choices with a clear ML upgrade path (Isolation Forest, time-series forecasting)

---

## Roadmap Beyond MVP

- Real QR/RFID hardware integration (replacing simulated check-in/out)
- Isolation Forest / Local Outlier Factor for statistical anomaly detection as historical data grows
- Time-series forecasting (e.g., Prophet, ARIMA) once sufficient multi-season data is available
- Role-based auth (dispatcher, site manager, admin)
- Push/SMS notifications for overdue alerts in addition to email
