GET /Equipment

[
  {
    "equipment_id": "EQX1001",
    "type": "Excavator",
    "site_id": "S003",
    "status": "checked_out",
    "check_out_date": "2025-04-01",
    "check_in_date": null,
    "last_operator_id": "OP101",
    "engine_hours_day": 1.5,
    "idle_hours_day": 10,
    "operating_days": 15
  }
]

POST /rentals/check-out

Request
{
  "equipment_id": "EQX1001",
  "site_id": "S003",
  "operator_id": "OP101",
  "expected_return_date": "2025-04-16"
}

Response

{
  "rental_id": 45,
  "equipment_id": "EQX1001",
  "site_id": "S003",
  "check_out_date": "2025-04-01",
  "expected_return_date": "2025-04-16",
  "status": "checked_out"
}

POST /rentals/check-in

Request
{
  "rental_id": 45,
  "check_in_date": "2025-04-16"
}

Response

{
  "rental_id": 45,
  "equipment_id": "EQX1001",
  "check_in_date": "2025-04-16",
  "status": "checked_in"
}

GET /usage-logs/{equipment_id}

{
  "equipment_id": "EQX1001",
  "logs": [
    {
      "date": "2025-04-05",
      "engine_hours": 1.5,
      "idle_hours": 10,
      "fuel_usage_l": 12.4,
      "location": "S003"
    }
  ],
  "summary": {
    "total_engine_hours": 22.5,
    "total_idle_hours": 150,
    "operating_days": 15
  }
}

GET /alerts/overdue

[
  {
    "equipment_id": "EQX1004",
    "site_id": "S004",
    "expected_return_date": "2025-05-14",
    "days_overdue": 1,
    "last_operator_id": "OP106"
  }
]

GET /forecast


{
  "period": "2025-05",
  "predictions": [
    {
      "site_id": "S003",
      "equipment_type": "Excavator",
      "predicted_rentals": 4,
      "confidence_note": "Based on linear regression, limited historical data"
    }
  ]
}

POST /anomalies/scan

Request

{ "equipment_id": "EQX1002" }

Response

{
  "scanned_at": "2025-04-20T09:00:00Z",
  "equipment_checked": 7,
  "anomalies_found": 3
}

GET /anomalies

[
  {
    "id": 12,
    "equipment_id": "EQX1002",
    "rule_code": "UNASSIGNED_SITE",
    "severity": "high",
    "reason": "EQX1002 checked out with no site assigned",
    "detected_at": "2025-04-20T09:00:00Z",
    "resolved": false
  },
  {
    "id": 13,
    "equipment_id": "EQX1002",
    "rule_code": "ZERO_USAGE",
    "severity": "medium",
    "reason": "EQX1002 shows 0 engine hours/day over 20 days",
    "detected_at": "2025-04-20T09:00:00Z",
    "resolved": false
  }
]