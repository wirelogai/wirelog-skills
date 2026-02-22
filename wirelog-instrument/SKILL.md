---
name: wirelog-instrument
description: Add WireLog event tracking and user identification to code
version: 1.0.0
---

# WireLog Instrumentation Skill

Add analytics event tracking and user identification to any application using WireLog's HTTP API.

## API Keys

- `pk_` (public key): Safe for client-side code. Can only ingest events.
- `sk_` (secret key): Full access (query + ingest). Keep server-side only.
- `aat_` (access token): Scoped permissions (`track`, `query`, `admin`).

## Track Events

```
POST /track
Header: X-API-Key: <pk_ or sk_ or aat_ with track scope>
Content-Type: application/json
```

### Single Event

```json
{
  "event_type": "page_view",
  "user_id": "u_123",
  "device_id": "dev_abc",
  "session_id": "sess_xyz",
  "time": "2026-01-15T10:30:00Z",
  "event_properties": {"page": "/pricing", "referrer": "google"},
  "user_properties": {"plan": "pro"},
  "insert_id": "dedup-id-optional"
}
```

Only `event_type` is required. All other fields are optional. `time` defaults to server time.

### Batch (up to 2000 events)

```json
{
  "events": [
    {"event_type": "page_view", "user_id": "u_123", "event_properties": {"page": "/home"}},
    {"event_type": "click", "user_id": "u_123", "event_properties": {"button": "signup"}}
  ]
}
```

### Response

```json
{"accepted": 2}
```

Invalid events are silently skipped. `accepted` is the count of valid events.

## Identify Users

Bind a `device_id` to a `user_id` and update the user profile. No event is emitted.

```
POST /identify
Header: X-API-Key: <pk_ or sk_ or aat_ with track scope>
Content-Type: application/json
```

```json
{
  "user_id": "alice@acme.org",
  "device_id": "dev_abc",
  "user_properties": {"email": "alice@acme.org", "plan": "pro"},
  "user_property_ops": {
    "$set": {"plan": "pro"},
    "$set_once": {"signup_source": "ads"},
    "$add": {"login_count": 1},
    "$unset": ["legacy_flag"]
  }
}
```

- `user_id` is required. `device_id` is recommended for identity stitching.
- `user_properties`: flat key-value set (overwrites).
- `user_property_ops`: granular operations (`$set`, `$set_once`, `$add`, `$unset`).

### Response

```json
{"ok": true}
```

## Identity Semantics

- Send `device_id` on every event and identify call.
- Call `/identify` when a user is known (login, signup, account link).
- Stitched identity: `distinct_id = coalesce(user_id, mapped_user_id, device_id)`.
- Pre-identify anonymous events are attributed after the device is identified.

## Recommended Profile Fields

For strong B2B/B2C analysis, send these via `/identify`:

- `email`, `plan`
- B2B: `company_id`, `company`, `account_tier`
- B2C: `acquisition_channel`, `persona`, `country`

## JS SDK (Browser)

```html
<script src="https://wirelog.ai/public/wirelog.js"
        data-key="pk_YOUR_PUBLIC_KEY"
        data-host="https://wirelog.ai"></script>
```

The SDK auto-tracks page views, manages `device_id` (localStorage), `session_id` (30-min timeout), and batches events (flush every 2s or 10 events).

```javascript
// Track an event
wl.track("button_click", {button: "signup", page: "/pricing"});

// Identify a user (call on login/signup)
wl.identify("alice@acme.org", {email: "alice@acme.org", plan: "pro"});

// Manual flush
wl.flush();
```

## Server-Side Examples

### curl

```bash
# Track
curl -X POST https://wirelog.ai/track \
  -H "X-API-Key: pk_YOUR_PUBLIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{"event_type":"signup","user_id":"u_123","event_properties":{"plan":"free"}}'

# Identify
curl -X POST https://wirelog.ai/identify \
  -H "X-API-Key: pk_YOUR_PUBLIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id":"u_123","device_id":"dev_abc","user_properties":{"email":"u_123@acme.org"}}'
```

### Python (stdlib)

```python
import json
from urllib.request import Request, urlopen

def track(event_type, user_id=None, props=None):
    body = {"event_type": event_type}
    if user_id:
        body["user_id"] = user_id
    if props:
        body["event_properties"] = props
    req = Request(
        "https://wirelog.ai/track",
        data=json.dumps(body).encode(),
        headers={"Content-Type": "application/json", "X-API-Key": "pk_YOUR_KEY"},
        method="POST",
    )
    with urlopen(req) as resp:
        return json.loads(resp.read())

track("signup", "u_123", {"plan": "free"})
```

### Node.js (fetch)

```javascript
await fetch("https://wirelog.ai/track", {
  method: "POST",
  headers: {"Content-Type": "application/json", "X-API-Key": "pk_YOUR_KEY"},
  body: JSON.stringify({event_type: "signup", user_id: "u_123", event_properties: {plan: "free"}}),
});
```
