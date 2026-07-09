# API Documentation

## Base URL

```
http://<server-address>:<port>/api/v1
```

---

## Endpoints

### 1. Submit Water Meter Reading

`POST /readings`

Submit a new water consumption reading from the meter.

**Request Body:**

```json
{
  "device_id": "string",
  "timestamp": "ISO 8601",
  "pulse_count": 1234,
  "flow_rate_lpm": 12.5,
  "volume_liters": 250.0,
  "total_liters": 10000.0
}
```

| Field          | Type   | Description                          |
|----------------|--------|--------------------------------------|
| device_id      | string | Unique identifier for the meter      |
| timestamp      | string | ISO 8601 datetime of the reading     |
| pulse_count    | int    | Raw pulse count in this interval     |
| flow_rate_lpm  | float  | Instantaneous flow rate (L/min)      |
| volume_liters  | float  | Volume since last reading            |
| total_liters   | float  | Cumulative total volume              |

**Response:**

```json
{
  "status": "ok",
  "reading_id": "abc123",
  "message": "Reading recorded"
}
```

---

### 2. Get Latest Reading

`GET /readings/latest?device_id={device_id}`

**Response:**

```json
{
  "device_id": "meter-001",
  "timestamp": "2026-07-10T01:00:00Z",
  "flow_rate_lpm": 10.2,
  "total_liters": 15000.0,
  "battery_pct": 85
}
```

---

### 3. Get Reading History

`GET /readings?device_id={device_id}&from={iso_date}&to={iso_date}&limit=100`

**Response:**

```json
{
  "device_id": "meter-001",
  "readings": [
    {
      "timestamp": "2026-07-10T00:00:00Z",
      "volume_liters": 1.5,
      "total_liters": 14998.5
    }
  ],
  "total": 1
}
```

---

### 4. Register Device

`POST /devices`

**Request:**

```json
{
  "device_id": "meter-001",
  "name": "Main Line Meter",
  "location": "Ground Floor"
}
```

**Response:**

```json
{
  "status": "ok",
  "api_key": "sk-xxxxxxxxxxxx"
}
```

---

### 5. Device Status / Health

`GET /devices/{device_id}/status`

**Response:**

```json
{
  "device_id": "meter-001",
  "online": true,
  "last_seen": "2026-07-10T01:00:00Z",
  "signal_rssi": -65,
  "battery_pct": 85,
  "firmware": "v1.2.0"
}
```

---

## Error Codes

| Code  | Meaning                    |
|-------|----------------------------|
| 200   | OK                         |
| 400   | Bad Request / Invalid JSON |
| 401   | Unauthorized / Bad API Key |
| 404   | Device Not Found           |
| 429   | Rate Limit Exceeded        |
| 500   | Internal Server Error      |

## Authentication

Include API key in header:

```
Authorization: Bearer sk-xxxxxxxxxxxx
```
