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
Header: X-API-Key: <personal aat_ query token or sk_>
Content-Type: application/json

{"q": "<query>", "format": "llm"}
```

Output formats: `llm` (default, Markdown tables), `json`, `csv`.

## Query Syntax

```
source | stage | stage | ...
```

### Sources

- Event name: `page_view`, `signup`, `landing:cta_click`, `docs.view`, `checkout/success`, etc. (use YOUR project's event names)
- Quoted event name: `"<event_name>"` (useful for special characters or ambiguity)
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
- Event inspection: `inspect *` (event type overview with counts, users, top properties) or `inspect <event>` (property-level detail: coverage %, type, sample values)
- Choice source: `choice <choice_key>` for client-side `wirelog.choice()` results

### Stages

- Filter: `| where field = "value"`, `| where field > 10`, `| where field contains "x"`, `| where field ~ "regex"`
- Operators: `=`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `not contains`, `~` (regex), `!~`, `in (...)`, `in [...]`, `not in (...)`, `not in [...]`, `exists`, `not exists`
- Boolean groups: `| where (a = "x" or b = "y") and c = "z"` (up to 32 nested parenthesized groups; filters remain inside the authenticated project scope)
- Time: `| last 7d`, `| last 12w`, `| from 2026-01-01 to 2026-02-01`, `| today`, `| yesterday`, `| this week`, `| this month`, `| this quarter`, `| this year`
- Aggregation: `| count`, `| unique distinct_id`, `| sum event_properties.amount`, `| avg field`, `| min field`, `| max field`, `| median field`, `| p90 field`, `| p95 field`, `| p99 field`
- Latest value per entity: `| latest event_properties.theme [per distinct_id]`; aggregate with `count by last_value` or list `entity`, `last_value`, `set_at`
- Group by: `| count by day`, `| count by week, _browser`, `| unique distinct_id by user.email_domain`
- List: `| list` (raw event rows)
- Sort: `| sort field desc`, `| sort count asc`
- Limit: `| limit 100`, `| top 20` (maximum 10,000 rows)
- Paths/funnel: `| window 7d`, `| depth 8`
- Choices: `| results <conversion_event>`, `| window 7d`, `| unit user_id`.

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

### Event inspection

Discover event types and their properties — the recommended first query for new projects:

```
inspect * | last 30d
inspect signup | last 7d
```

`inspect *` returns per-event-type overview: count, unique users, first/last seen, top property keys.
`inspect <event>` returns property-level detail: coverage %, inferred type (string/numeric/boolean), sample values.

### Identity

- `distinct_id` = stitched identity: `coalesce(user_id, mapped_user_id, device_id)`.
- Use `unique distinct_id` for unique user counts.
- Pre-identify anonymous events are attributed once device-to-user mapping exists.
- For durable current user state, prefer `identify`/profile properties and query `users | count by user.theme`.
- For event-only setter/change history, use `themeSwitch | latest event_properties.theme | count by last_value | top 10`.

## IMPORTANT: Discover Events First

Event names are NOT hardcoded. Start with inspect to understand the project's data:

```
inspect * | last 30d
```

Then inspect specific events to see their properties before writing queries:

```
inspect signup | last 7d
```

Alternative discovery (counts only):

```
* | last 30d | count by event_type | top 20
```

## Choices

Use this workflow when asked to analyze a `wirelog.choice()` experiment:

```
wl query "inspect * | last 30d" --json
wl choice results landing_h1 --conversion signup --window 7d
wl query "choice landing_h1 | results signup | window 7d | unit user_id" --json
```

When running multiple independent checks, prefer one CLI call with repeated
`--query` flags instead of asking the shell to manage several commands:

```
wl query --query "inspect * | last 30d" --query "* | last 30d | count by event_type | top 20" --json
```

Choices are declared in application code with `choice()`, resolve
synchronously, and record exposures asynchronously. Use `unit user_id` for the
usual product experiment case and `unit device_id` for anonymous visitor tests.

Useful analysis queries:

```
choice landing_h1 | results signup | window 7d | unit user_id
choice landing_h1 | last 7d | count by variant_key
```

If your project only has the Script Tag installed, you'll usually see a lot of `page_view` plus any custom frontend events. Start with `page_view` queries first, then branch out.

## Agent-Created Dashboards

Use dashboards when the user asks for a shareable report, local dashboard, or many related WireLog queries in one view.

Dashboards are YAML files agents can create, validate, run, view, and export:

```
wl dashboard schema --output -
wl dashboard init --output -
wl dashboard validate --file dashboard.yaml
wl dashboard validate --file - --json
wl dashboard run --file dashboard.yaml --json
wl dashboard run --file dashboard.yaml --var range=7d --format markdown
wl dashboard view --file dashboard.yaml --open
wl dashboard view --file ./dashboards
```

Export modes:

- `report`: fixed data, no key embedded. Prefer this for sharing.
- `interactive`: embeds a query-scoped `aat_` token so controls can re-query from the browser. Team members should use their own personal query token. Never use `sk_`, `pk_`, or `ak_`.

```
wl dashboard save --file dashboard.yaml --output index.html --mode report
wl dashboard save --file dashboard.yaml --output - --mode report
wl dashboard save --file dashboard.yaml --output index.html --mode interactive --token-env WIRELOG_DASHBOARD_TOKEN
```

Interactive exports written to files use `0600` permissions because the HTML contains the token.
Use `wl dashboard view --file <dir>` for a dashboard directory; the UI renders a sidebar for `.yaml` and `.yml` files.
Directory dashboards have stable local routes like `/dashboard/usage.yaml`; extensionless routes like `/dashboard/usage` work when unambiguous.

Dashboard root fields:
- `order: 10` controls directory sidebar order; leave gaps like 10, 20, 30.
- `timezone: UTC` controls display timezone; use the user's preferred IANA timezone when known.
- `refresh: 60s` sets default live refresh.

Start every dashboard from discovery:

```
wl query --query "inspect * | last 30d" --query "* | last 30d | count by event_type | top 20" --json
```

Dashboards automatically get a built-in `range` date-range control unless they define `variables.range`. Use `{{range.stage}}` to insert a full time stage; use variables for shared segments:

```yaml
version: 1
title: Product Growth
order: 10
refresh: 60s
timezone: UTC

variables:
  range:
    label: Range
    type: date_range
    default: 30d
  platform:
    label: Platform
    type: select
    default: all
    options:
      - label: All
        value: all
        fragment: ""
      - label: Web
        value: web
        fragment: '| where _platform = "web"'

sections:
  - title: Acquisition
    cards:
      - id: acquisition-trends
        title: Acquisition Trends
        kind: chart
        viz: line
        layout: {w: 12, h: 4}
        queries:
          - name: Signups
            query: signup {{range.stage}} {{platform.fragment}} | count by day
          - name: Activations
            query: activate {{range.stage}} {{platform.fragment}} | count by day
```

Dynamic dropdowns can come from data:

```yaml
variables:
  country:
    label: Country
    type: select
    default: all
    options:
      - label: All
        value: all
        fragment: ""
    query: '* | last 30d | count by _country | top 25'
    value_column: _country
    label_column: _country
    fragment_template: '| where _country = "{{value}}"'
```

User lookup dashboards use submitted input variables with safe named fragments:

```yaml
variables:
  subject:
    label: User
    type: input
    input: email
    required: true
    submit: true
    placeholder: "email or *@example.com"
    allow_domain_wildcard: true
    fragments:
      events:
        exact_field: user.email
        domain_field: user.email_domain
      users:
        exact_field: email
        domain_field: email_domain

sections:
  - title: User
    cards:
      - id: user-profile
        title: Profile
        kind: users
        viz: table
        query: 'users {{subject.users_fragment}} | list'
      - id: recent-events
        title: Recent Events
        kind: events
        viz: event-stream
        query: '* {{subject.events_fragment}} {{range.stage}} | list | limit 100'
```

Use chart options when column inference could be ambiguous:

```yaml
options: {x: day, y: value, series: _browser}
query: 'page_view {{range.stage}} | count by day, _browser | top 50'
```

Line, area, and bar cards render time bucket columns on chronological axes and align multi-series buckets; missing bucket values display as gaps. Line and area charts keep the active bucket live, draw its final segment as dashed, and mark its tooltip `partial`. For grouped time queries, set `options.x`, `options.y`, and `options.series` explicitly.

Local and interactive dashboards progressively run visible cards in layout order, two at a time, so the first row can render before lower rows finish querying.

Dashboard-side ratios use two normal aggregate queries:

```yaml
options: {calculate: ratio, x: day, y: value}
queries:
  - name: Purchases
    query: purchase {{range.stage}} | count by day
  - name: Signups
    query: signup {{range.stage}} | count by day
```

Rules:

- Use real event names discovered from `inspect *`; do not invent project-specific events.
- Use `query` for one series and `queries` for overlays/comparisons.
- Use the built-in `{{range.stage}}` for dashboard-wide date windows; old `| last {{range}}` templates are accepted for compatibility.
- Use `select` variables for dropdowns; use `input` only with safe named fragments.
- Never splice raw user text into queries.
- Use `options.x`, `options.y`, and `options.series` when chart columns are ambiguous.
- Validate before saving or exporting.

## Script-Tag-First Starter Queries (page_view heavy)

Use these immediately after discovery when the dataset is mostly browser traffic:

```
1.  page_view | last 7d | count by day
    -- Daily traffic trend
2.  page_view | last 7d | unique distinct_id by day
    -- Daily unique visitors (stitched identity)
3.  page_view | last 30d | count by _path | top 20
    -- Top pages by path
4.  page_view | last 30d | count by _path_clean | top 20
    -- Top pages with dynamic IDs normalized
5.  page_view | where _path in ["/","/pricing","/docs"] | last 7d | count
    -- Focus on a specific set of paths (JSON-style list)
6.  page_view | where _path in ("/","/pricing","/docs") | last 7d | count
    -- Same as above (SQL-style list)
7.  page_view | last 30d | count by _browser | top 10
    -- Browser mix
8.  page_view | last 30d | count by _country | top 10
    -- Geographic mix
9.  paths from page_view | last 30d | by _path
    -- Navigation flow using URL paths as nodes
10. sessions | last 30d | count by session.utm_source
    -- Acquisition channels at session level
11. sessions | last 30d | count by session.referring_domain | top 20
    -- Top referring domains
12. retention page_view | last 90d
    -- Cohort retention from first visit
```

If event names include special characters, both direct source and quoted source forms work:

```
landing:cta_click | last 7d | count
"landing:cta_click" | last 7d | count
docs.view | last 7d | count
checkout/success | last 7d | count
```

Fallback form when in doubt:

```
* | where event_type = "landing:cta_click" | last 7d | count
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
26. inspect * | last 30d
    -- Discover all event types with counts, users, and top properties
27. inspect signup | last 7d
    -- Inspect a specific event's properties (coverage, types, sample values)
```

## Example curl

```bash
curl -X POST https://api.wirelog.ai/query \
  -H "X-API-Key: sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"q":"* | last 30d | count by event_type | top 20"}'
```

For `format: "json"` aggregate-style responses, treat these as canonical when present:
- `value`: numeric metric value
- `metric`: metric identifier (`count`, `unique`, `sum`, `avg`, etc.)

Legacy keys (`count`, `unique_count`, `total`, ...) may also be present for compatibility.
