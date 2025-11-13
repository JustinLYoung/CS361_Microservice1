# Universal Reminder & Notification Service (URNS)

## Overview
The **Universal Reminder & Notification Service (URNS)** is a standalone microservice that schedules and delivers reminders via webhooks.

It supports:
- **One-time reminders**
- **Recurring reminders** (cron syntax)
- **Webhook delivery callbacks**
- **Retry logic with exponential backoff**
- **API-key authentication**
- **In-memory storage** (Sprint 1)

This service is meant to be consumed by other microservices (Weather App, Calendar Service, User Service, etc.).

---

## 🚀 Running the URNS Microservice

### 1. Activate your virtual environment
```bash
python -m venv .venv
.venv\Scripts\activate
```

### 2. Install dependencies
If a `requirements.txt` exists inside `urns/`:
```bash
pip install -r requirements.txt
```

If your repo has a top-level `requirements.txt`:
```bash
pip install -r ../requirements.txt
```

### 3. Start the URNS server
From inside the `urns/` folder:
```bash
uvicorn app:app --reload --port 8081
```

URNS will now be running at:  
👉 http://127.0.0.1:8081

---

## 🧩 API Endpoints

### Health Check
```http
GET /healthz
→ {"status": "ok"}
```

---

### Create a One-Time Reminder
```http
POST /reminders
Headers:
  X-App-Key: dev-key
Body:
{
  "app_id": "weather-app",
  "type": "time",
  "when": "2025-11-12T07:00:00Z",
  "notify": { "webhook": "http://127.0.0.1:8080/hooks/reminder" },
  "payload": { "title": "Hello", "msg": "This is your reminder." }
}
```

---

### Create a Recurring Reminder (cron)
```http
POST /reminders
Headers:
  X-App-Key: dev-key
Body:
{
  "app_id": "weather-app",
  "type": "cron",
  "cron": "0 7 * * *",
  "notify": { "webhook": "http://127.0.0.1:8080/hooks/reminder" },
  "payload": { "title": "Daily Check", "msg": "Look at the weather today!" }
}
```

---

### List Reminders
```http
GET /reminders
Headers: X-App-Key: dev-key
```

### Get One Reminder
```http
GET /reminders/{reminder_id}
Headers: X-App-Key: dev-key
```

---

### Delete One Reminder
```http
DELETE /reminders/{reminder_id}
Headers: X-App-Key: dev-key
```

### Delete ALL Reminders
```http
DELETE /reminders
Headers: X-App-Key: dev-key
```

---

## 🔔 Webhook Delivery Format

When URNS sends a reminder, it POSTs this JSON to your webhook:

```json
{
  "reminder_id": "d1a92df3-1c89-4872-a94f-b3f41c3eb266",
  "app_id": "weather-app",
  "fired_at": "2025-11-12T07:00:00Z",
  "payload": {
    "title": "Daily Forecast",
    "msg": "Check the weather!"
  }
}
```

With headers:
```text
Content-Type: application/json
X-App-Id: weather-app
X-App-Key: dev-key
X-URNS-Delivery: 1
```

---

## ⚙️ Internal Behavior

### Storage
- Uses an in-memory dictionary (`REMINDERS`).
- Does not persist once the server stops (Sprint 1 requirement).

### Scheduler
URNS uses **APScheduler**:
- `DateTrigger` for one-time reminders
- `CronTrigger` for recurring reminders

### Delivery Retries
If a webhook returns an error:
- URNS retries **3 times**
- Backoff schedule: **2 seconds → 8 seconds → 30 seconds**
- One-time reminders eventually become `failed`
- Cron reminders wait until the next scheduled tick

---

## 🧠 Quality Attributes

### Testability
- Pure async logic  
- Easy to test using pytest + HTTPX test client  

### Availability
- Retry logic  
- Clear status fields (`scheduled`, `delivered`, `cancelled`, `failed`)  

### Maintainability
- Clean separation of:
  - validation
  - scheduling
  - delivery
  - API routes  

### Security
- API key authentication via `X-App-Key`
- Each webhook can require its own auth

---

# 🤝 Team Integration Notes

---

## 1. Universal Users Service
- Stores users in SQLite  
- Communicates with ZMQ  

How it works with URNS:
- A user’s reminders can be grouped using `app_id`  
- URNS does **not** store user records  
- The Users Service can call URNS when a user configures a reminder  

Example:
```json
{
  "app_id": "user:123",
  "type": "time",
  "when": "...",
  "notify": { "webhook": "http://127.0.0.1:5000/user/123/inbox" }
}
```

---

## 2. Calendar Microservice
- Accepts events  
- Generates timestamps  
- Uses ZMQ messaging  

How it integrates:
1. Calendar microservice outputs a timestamp  
2. Weather or User app passes that timestamp into URNS  
3. URNS schedules the actual reminder

Example:
```http
POST /reminders
{
  "app_id": "calendar",
  "type": "time",
  "when": "<timestamp from calendar service>",
  "notify": { "webhook": "http://127.0.0.1:8080/hooks/calendar" }
}
```

URNS becomes the **central scheduling engine** for all microservices.

