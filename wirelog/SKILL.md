---
name: wirelog
description: Query WireLog analytics using the pipe-based DSL. Returns Markdown tables, JSON, or CSV.
version: 1.0.0
---

# WireLog Query Skill

You can query analytics data from WireLog using a pipe-based DSL via HTTP.

## API

```
POST /query
Header: X-API-Key: <sk_ or aat_ with query scope>
Content-Type: application/json

{"q": "<query>", "format": "llm"}
```

Output formats: `llm` (default, Markdown tables), `json`, `csv`.

## Query Syntax

```
source | stage | stage | ...
```

### Sources

- Event name: `page_view`, `signup`, etc. (use YOUR project's event names)
- All events: `*`
- Funnel: `funnel signup -> activate -> purchase`
- Retention: `retention signup` or `retention signup returning return_event`
- Paths: `paths from event` or `paths to event`
- Sessions: `sessions`
- Single user timeline: `user "alice@acme.org"`
- Users directory: `users`
- Formula: `formula count(purchase) / count(signup)`

### Stages

- Filter: `| where field = "value"`, `| where field > 10`, `| where field contains "x"`
- Time: `| last 7d`, `| last 12w`, `| from 2026-01-01 to 2026-02-01`, `| today`, `| this month`
- Aggregation: `| count`, `| unique distinct_id`, `| sum event_properties.amount`, `| avg field`, `| min field`, `| max field`, `| median field`, `| p90 field`, `| p95 field`, `| p99 field`
- Group by: `| count by day`, `| count by week, _browser`, `| unique distinct_id by user.email_domain`
- List: `| list` (raw event rows)
- Sort: `| sort field desc`, `| sort count asc`
- Limit: `| limit 100`, `| top 20`

### Fields

- Core: `event_type`, `user_id`, `distinct_id`, `device_id`, `session_id`, `time`
- System: `_browser`, `_os`, `_platform`, `_device_type`, `_ip`
- Event properties: `event_properties.KEY`
- User properties (on event): `user_properties.KEY`
- Profile fields: `user.email`, `user.email_domain`, `user.plan`, `user.first_seen`, `user.last_seen`, `user.KEY`

### Identity

- `distinct_id` is stitched identity: `coalesce(user_id, mapped_user_id, device_id)`.
- Use `unique distinct_id` for unique user counts.
- Pre-identify anonymous events are attributed once device-to-user mapping exists.

## IMPORTANT: Discover Event Names First

Event names are NOT hardcoded. Run these before writing event-specific queries:

```
* | last 30d | count by event_type | top 20
```

Full inventory:

```
* | last 90d | count by event_type | sort event_type asc | limit 10000
```

## Example Queries

Replace placeholders with your real event names.

```
1.  <event_name> | last 7d | count
2.  <event_name> | last 30d | count by day
3.  * | last 24h | count by event_type
4.  <event_name> | where _platform = "web" | last 30d | count
5.  <signup_event> | last 90d | unique distinct_id by week
6.  funnel <signup_event> -> <activation_event> -> <purchase_event> | last 30d
7.  retention <signup_event> | last 90d
8.  paths from <start_event> | last 30d | depth 8
9.  sessions | last 7d
10. users | where email_domain = "acme.org" | list
11. * | where user.email_domain = "acme.org" | last 30d | count by event_type
12. user "alice@acme.org" | last 90d | list
13. formula count(<purchase_event>) / count(<signup_event>) | last 30d
14. <core_usage_event> | where user.plan = "enterprise" | last 12w | count by week
15. * | last 12w | count by user.email_domain | sort count desc | top 20
```

## Example curl

```bash
curl -X POST https://api.wirelog.ai/query \
  -H "X-API-Key: sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"* | last 30d | count by event_type | top 20"}'
```
