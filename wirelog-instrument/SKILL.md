---
name: wirelog-instrument
description: Add WireLog event tracking and user identification to code
version: 1.0.0
---

# WireLog Instrumentation Skill

Add analytics event tracking and user identification to any application using WireLog.

For latest documentation: https://docs.wirelog.ai/llms.txt

## Recommended SDKs

Use the official SDKs whenever possible. They handle batching, retries, identity management, and session tracking automatically.

### Browser (JS SDK script tag)

```html
<script src="https://cdn.wirelog.ai/public/wirelog.js"
        data-key="pk_YOUR_PUBLIC_KEY"
        data-host="https://api.wirelog.ai"></script>
```

Auto-tracks page views, manages `device_id`/`session_id`, batches events, and sets `clientOriginated: true`.

```javascript
wl.track("button_click", {button: "signup", page: "/pricing"});
wl.identify("alice@acme.org", {email: "alice@acme.org", plan: "pro"});
wl.flush();
```

### TypeScript / Node.js (npm)

```bash
npm install wirelog
```

```typescript
import { wl } from "wirelog";

wl.init({ apiKey: "pk_YOUR_PUBLIC_KEY" });

wl.track({ event_type: "signup", user_id: "u_123", event_properties: { plan: "free" } });
wl.identify({ user_id: "u_123", user_properties: { name: "Alice", plan: "pro" } });
```

### Raw HTTP (any language)

If no SDK is available for your language, use the HTTP API directly:

```
POST /track
Header: X-API-Key: <pk_ or sk_ or aat_ with track scope>
Content-Type: application/json
```

## API Keys

- `pk_` (public key): Safe for client-side code. Can only ingest events.
- `sk_` (secret key): Full access (query + ingest). Keep server-side only.
- `aat_` (access token): Scoped permissions (`track`, `query`, `admin`).

## What to Track

Choose events that map to your product's key actions. Here are recommended events by use case:

### SaaS / Web App
- `page_view` (auto-tracked by JS SDK)
- `signup`, `login`, `logout`
- `feature_used` with `event_properties.feature` (e.g., "export", "invite", "search")
- `subscription_started`, `subscription_cancelled`, `payment_completed`
- `invite_sent`, `team_member_added` (B2B)

### E-Commerce
- `page_view`, `search`, `item_viewed`, `add_to_cart`, `checkout_started`, `purchase`
- `coupon_applied`, `review_submitted`

### AI Agent / LLM App
- `agent_action` with `event_properties.action` (e.g., "query", "tool_call", "response")
- `agent_error` with `event_properties.error_type`
- `token_usage` with `event_properties.tokens`
- `user_feedback` with `event_properties.rating`

### General Best Practices
- Use `event_properties` for event-specific data (what happened)
- Use `user_properties` / `/identify` for user-level data (who they are)
- Send `device_id` on every event for identity stitching
- Keep event names snake_case and descriptive

## Track Events (HTTP API)

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

### Origin Hints

- Browser events: set `clientOriginated: true` (JS SDK does this automatically)
- Server-side events: set `origin: "server"`
- Origin hints improve session attribution and geo enrichment accuracy

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
- Response: `{"ok": true}`

### Recommended Profile Fields

For strong B2B/B2C analysis, send these via `/identify`:

- `email`, `plan`
- B2B: `company_id`, `company`, `account_tier`
- B2C: `acquisition_channel`, `persona`, `country`

## Identity Semantics

- Send `device_id` on every event and identify call.
- Call `/identify` when a user is known (login, signup, account link).
- Stitched identity: `distinct_id = coalesce(user_id, mapped_user_id, device_id)`.
- Pre-identify anonymous events are attributed after the device is identified.
