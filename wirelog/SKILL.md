---
name: wirelog
description: Query WireLog analytics using the pipe-based DSL. Returns Markdown tables, JSON, or CSV.
version: 1.0.0
---

# WireLog Query Skill

Query analytics data from WireLog using a pipe-based DSL via HTTP.

For latest documentation: https://docs.wirelog.ai/llms.txt

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
- Funnel with exclusion: `funnel signup -> purchase exclude support_ticket` (between-step exclusion, NOT global NOT IN)
- Retention: `retention signup` or `retention signup returning return_event` or `retention signup returning any`
- Paths: `paths from event` or `paths to event` (supports `| by <field>` to label path nodes by a field instead of event_type)
- Lifecycle: `lifecycle active_user`
- Stickiness: `stickiness core_usage`
- Sessions: `sessions`
- Single user timeline: `user "alice@acme.org"`
- Users directory: `users`
- Formula: `formula count(purchase) / count(signup)`
- Fields introspection: `fields` (lists all available fields including dynamic property keys)

### Stages

- Filter: `| where field = "value"`, `| where field > 10`, `| where field contains "x"`, `| where field ~ "regex"`
- Operators: `=`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `not contains`, `~` (regex), `!~`, `in (...)`, `not in (...)`, `exists`, `not exists`
- Boolean groups: `| where (a = "x" or b = "y") and c = "z"` (nested parentheses supported)
- Time: `| last 7d`, `| last 12w`, `| from 2026-01-01 to 2026-02-01`, `| today`, `| yesterday`, `| this week`, `| this month`, `| this quarter`, `| this year`
- Aggregation: `| count`, `| unique distinct_id`, `| sum event_properties.amount`, `| avg field`, `| min field`, `| max field`, `| median field`, `| p90 field`, `| p95 field`, `| p99 field`
- Group by: `| count by day`, `| count by week, _browser`, `| unique distinct_id by user.email_domain`
- List: `| list` (raw event rows)
- Sort: `| sort field desc`, `| sort count asc`
- Limit: `| limit 100`, `| top 20`
- Paths/funnel: `| window 7d`, `| depth 8`

### Fields

- Core: `event_type`, `user_id`, `distinct_id`, `device_id`, `session_id`, `time`
- Page/Content: `_url`, `_path`, `_path_clean` (ID-normalized path), `_hostname`, `_title` (promoted from event properties during enrichment)
- Referrer: `_referrer`, `_referrer_domain` (promoted from event properties during enrichment)
- Device: `_browser`, `_browser_version`, `_os`, `_os_version`, `_platform`, `_device_type`
- Geo: `_country`, `_city`, `_region`, `_continent`, `_timezone` (short aliases for `_geo_*` fields)
- Geo (full): `_geo_source`, `_geo_country`, `_geo_region`, `_geo_region_code`, `_geo_city`, `_geo_continent`, `_geo_timezone`, `_geo_metro_code`, `_geo_postal_code`, `_geo_latitude`, `_geo_longitude`
- Other system: `_ip`, `_ua`, `_library`, `_ingest_origin`
- Event properties: `event_properties.KEY`
- User properties (on event): `user_properties.KEY`
- Profile fields: `user.email`, `user.email_domain`, `user.plan`, `user.first_seen`, `user.last_seen`, `user.KEY`
- Session fields: `session.start_time`, `session.end_time`, `session.duration`, `session.event_count`, `session.landing_url`, `session.landing_path`, `session.landing_path_clean`, `session.landing_hostname`, `session.referrer`, `session.referring_domain`, `session.utm_source`, `session.utm_medium`, `session.utm_campaign`, `session.utm_term`, `session.utm_content`, `session.gclid`, `session.fbclid`, `session.language`, `session.timezone`, `session.ingest_origin`, `session.geo_source`, `session.country`, `session.region`, `session.region_code`, `session.city`, `session.continent`, `session.geo_timezone`, `session.metro_code`, `session.postal_code`, `session.latitude`, `session.longitude`, `session.first_event`, `session.last_event`, `session.exit_url`, `session.exit_path`, `session.exit_path_clean`
- Latest stitched user-session fields: `user_last_session.<same keys as session.*>` (resolves to each user's most recent stitched session)

### Field aliases

Short aliases resolve to canonical field names at compile time: `_country` → `_geo_country`, `_city` → `_geo_city`, `_region` → `_geo_region`, `_continent` → `_geo_continent`, `_timezone` → `_geo_timezone`, `_page` → `_path`, and more.

### Field introspection

Discover all available fields at runtime:

```
fields | last 7d
```

Returns system/core fields plus dynamic `event_properties.*` and `user_properties.*` keys actually present in your data.

### Identity

- `distinct_id` = stitched identity: `coalesce(user_id, mapped_user_id, device_id)`.
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
    -- How many times did this event fire in the last week?
2.  <event_name> | last 30d | count by day
    -- Daily trend for this event
3.  * | last 24h | count by event_type | top 20
    -- What's happening right now? Top events by volume
4.  <event_name> | where _platform = "web" | last 30d | count
    -- Filter to a specific platform
5.  <signup_event> | last 90d | unique distinct_id by week
    -- Weekly unique signups (stitched identity)
6.  funnel <signup_event> -> <activation_event> -> <purchase_event> | last 30d
    -- Conversion funnel: what % of signups become paying customers?
7.  funnel <signup_event> -> <purchase_event> exclude <support_event> | window 7d
    -- Funnel with between-step exclusion: disqualify if support_event occurs between steps
8.  retention <signup_event> | last 90d
    -- Week-over-week retention of signup cohort
9.  retention <signup_event> returning <core_usage_event> | last 90d
    -- Do signups come back and use the core feature?
10. paths from <start_event> | last 30d | window 7d | depth 8
    -- What do users do after this event? (shows event_type names by default)
10b. paths from page_view | last 30d | by _path
    -- Navigation flow by URL path (e.g. /home -> /pricing -> /signup)
10c. paths from page_view | last 30d | by _path_clean
    -- Navigation flow with IDs normalized (e.g. /users/{id}/settings instead of /users/123/settings)
10d. page_view | last 30d | count by _path_clean | top 20
    -- Top pages with dynamic IDs grouped (UUIDs, numeric IDs, etc. become {id})
11. sessions | where session.utm_source = "google" | last 30d | count by day
    -- Session volume from Google by day
12. lifecycle page_view | last 12w | by week
    -- New / Returning / Resurrected / Dormant user segments
13. stickiness page_view | last 30d
    -- How many days per period are users active?
14. users | where email_domain = "acme.org" | list
    -- Look up all users from a specific company
15. * | where user.email_domain = "acme.org" | last 30d | count by event_type
    -- What is this company doing in your product?
16. user "alice@acme.org" | last 90d | list
    -- Full timeline for a single user
17. formula count(<purchase_event>) / count(<signup_event>) | last 30d
    -- Conversion rate as a ratio
18. sessions | last 30d | count by session.utm_source
    -- Which channels are driving sessions?
19. sessions | last 30d | count by session.referring_domain | top 10
    -- Top referring domains
20. <spend_event> | where user_last_session.region = "DE" | last 30d | sum event_properties.amount
    -- Revenue from users whose last browser session was in Germany
21. * | where (session.utm_source = "google" or session.utm_source = "bing") and _device_type = "mobile" | last 30d | count
    -- Boolean grouping with parentheses
22. <event_name> | last 30d | count by _path | top 20
    -- Top pages by URL path
23. <event_name> | last 30d | count by _country | top 10
    -- Events by country (short alias for _geo_country)
24. sessions | last 30d | count by session.landing_path | top 20
    -- Top landing page paths
25. fields | last 7d
    -- Discover all available fields (system + dynamic property keys)
```

## Example curl

```bash
curl -X POST https://api.wirelog.ai/query \
  -H "X-API-Key: sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"* | last 30d | count by event_type | top 20"}'
```
