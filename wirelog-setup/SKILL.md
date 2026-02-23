---
name: wirelog-setup
description: Set up a new WireLog analytics project from scratch
version: 1.0.0
---

# WireLog Setup Skill

Set up a new WireLog analytics project. This skill covers account creation, project setup, API key retrieval, and verification.

## Overview

WireLog is headless analytics for AI agents and LLMs. You track events via HTTP, query with a pipe DSL, and get Markdown back. No dashboards — your agent is the dashboard.

**Hierarchy:** User → Organization → Project(s). Each project gets a `pk_` (public) and `sk_` (secret) key.

## Step 1: Sign Up

Go to https://wirelog.ai and sign in with GitHub. This creates your user and default org automatically.

## Step 2: Create a Project

### Via Web UI

1. Go to https://wirelog.ai/dashboard
2. Click "+ New Project"
3. Name it (e.g., `my-app-prod`)
4. Copy the `pk_` and `sk_` keys shown

### Via Admin API

If you have an org admin key (`ak_`):

```bash
curl -X POST https://api.wirelog.ai/api/admin/projects \
  -H "X-API-Key: ak_YOUR_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-app-prod"}'
```

Response includes `public_key` (pk_) and `secret_key` (sk_). Store the secret key securely — it's only shown once.

## Step 3: Add Tracking

### Browser (JS SDK)

Add to your HTML `<head>`:

```html
<script src="https://cdn.wirelog.ai/public/wirelog.js"
        data-key="pk_YOUR_PUBLIC_KEY"
        data-host="https://api.wirelog.ai"></script>
```

This auto-tracks page views and manages device/session IDs.

### Server-Side (any language)

```bash
curl -X POST https://api.wirelog.ai/track \
  -H "X-API-Key: pk_YOUR_PUBLIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{"event_type":"test_event","user_id":"setup-test","event_properties":{"source":"setup-verification"}}'
```

The public key (`pk_`) is safe for client-side code — it can only ingest events.

## Step 4: Verify

Query to confirm events are flowing:

```bash
curl -X POST https://api.wirelog.ai/query \
  -H "X-API-Key: sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"* | last 1d | count by event_type"}'
```

You should see your `test_event` (or page views from the JS SDK) in the Markdown table response.

## Step 5: Set Up Identity (Recommended)

When a user logs in or signs up, call identify to bind their device to their user ID:

```bash
curl -X POST https://api.wirelog.ai/identify \
  -H "X-API-Key: pk_YOUR_PUBLIC_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "alice@acme.org",
    "device_id": "dev_abc",
    "user_properties": {"email": "alice@acme.org", "plan": "free"}
  }'
```

This enables identity-stitched analytics (`distinct_id`) across anonymous and known sessions.

## Key Reference

| Key prefix | Access level | Safe for client-side? |
|---|---|---|
| `pk_` | Track only (ingest events) | Yes |
| `sk_` | Full access (track + query) | No — server-side only |
| `ak_` | Org admin (manage projects) | No — server-side only |
| `aat_` | Scoped access token | Depends on granted scopes |

## What's Next

- Discover your events: `* | last 30d | count by event_type | top 20`
- Build funnels: `funnel signup -> activate -> purchase | last 30d`
- Track retention: `retention signup | last 90d`
- Segment by company: `* | where user.email_domain = "acme.org" | last 30d | count by event_type`
- See query-language docs for full DSL reference
