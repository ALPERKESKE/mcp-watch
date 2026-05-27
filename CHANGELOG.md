# Changelog

All notable changes to **MCP-Watch for Splunk** are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/), and this project
adheres to [Semantic Versioning](https://semver.org/).

---

## [1.1.0] — 2026-05-27

First update since the Splunkbase release of 1.0.0. Bundles two internal
development phases (the *multi-signal detection* phase and the *liveness
heartbeats* phase) into a single Splunkbase minor release.

### Added

#### Multi-signal MCP detection (custom / community / federation MCPs)
- v1.0 only saw clients carrying the official provenance stamp
  (`MCP:Splunk_MCP_Server:*`). v1.1 also recognises custom / community
  MCP servers and federation gateways via **REST-side user-agent matching**
  (`mcp_user_agents.csv` lookup) and **endpoint-based tool inference**.
- New **MCP Detection (REST)** dashboard showing: MCP / agent clients
  detected by user-agent, REST activity by inferred tool, top REST
  endpoints from non-official clients, and detection signal coverage
  (official-provenance vs. user-agent-only).
- Conservative default user-agent patterns (generic `python-httpx*` is
  opt-in) to avoid flagging non-MCP automation.

#### Unified MCP Access & Tools dashboard
- Works whether or not the official Splunk MCP Server is present. With the
  official server, uses the `mcp_tool_execute` capability + tool catalog;
  otherwise derives access from the connecting account's **RBAC** +
  **REST-detected tool usage**.
- Custom MCPs no longer leave the dashboard blank.
- New lookups: `mcp_tool_catalog.csv` (14-tool catalog),
  `mcp_tool_denied.csv` (per-user deny policy — governance/intent layer,
  see *Access model* in README).
- Empty-safe `where isnotnull(user)` guard on the User × Tool matrix.

#### Getting Started dashboard
- In-app onboarding showing current configuration, how to fill
  `mcp_users.csv`, and live "is data flowing?" checks. Open this first
  after install.

#### MCP liveness heartbeats with UP / IDLE / STALE
- New `mcp_heartbeat` KV collection. MCP Overview's top row now shows
  per-MCP status with a three-state model layering heartbeat freshness
  with recent-activity:
  - **`● UP`** — heartbeat ≤ 6 min old **AND** MCP queries in the last hour.
  - **`◐ IDLE`** — heartbeat fresh but no MCP queries in the last hour
    (e.g. IDE open in background, no active conversation).
  - **`○ STALE`** — no fresh heartbeat. The MCP server itself is down.
- Two-state (UP/STALE) would have shown an idle agent as UP, hiding the
  difference between "everything's fine but nothing's happening" and
  "agent actively working" — useful for triaging "AI side broken" vs
  "MCP server broken".
- **Auto-heartbeat for the official Splunk MCP Server**: new scheduled
  report `MCP-Watch - Heartbeat - Official MCP Server` writes a heartbeat
  every 5 min when the official server app is enabled. No setup needed
  for the primary use case.
- Custom / external MCPs send their own heartbeat (the MCP runs outside
  Splunk, so it is the only reliable liveness signal) — see README for
  the `batch_save` example.

#### `is_export_or_delete` flag + CRITICAL-severity alert
- New anti-pattern: `| outputcsv`, `| outputlookup`, `| collect`,
  `| sendemail`, or `| delete` in MCP SPL → flag fires with **weight
  +15** (single fire reaches CRITICAL band on its own).
- This flag is categorically different from the others: it captures
  *agent intent* to write data out (exfil) or destroy data, not "sloppy
  SPL". For an MCP service account these commands are almost never
  legitimate — scheduled exports/backups are owned by humans via saved
  searches, and `| delete` needs the `delete_by_keyword` capability
  that an `mcp_agent` role should not have.
- Paired with a dedicated alert
  **`MCP-Watch - Alert - Export or Delete Command Used`** at
  **severity 5 (CRITICAL)**, 15-min suppression. Investigates the source
  agent and the SPL body immediately on a hit.
- Self-test fixtures added in `lookups/regex_fixtures.csv` (10 rows
  covering all five command variants + 3 negative cases).

#### Splunkbase metadata polish
- App icons added (36×36 + 72×72; `appIcon` / `appIconAlt`).
- Support contact (`alperkeske@gmail.com`) in `app.manifest` and README.
- Disclaimers added to README: personal project (no employer / Splunk
  affiliation) and AI-assisted development disclosure.
- `check_for_updates = true` in `app.conf` (Splunkbase requires it).

#### Reserved-user exclusion (`mcp_excluded_users.csv`)
- New lookup centralises the accounts that discovery panels and the REST
  detection macro should never treat as MCP agents. Ships with `admin`,
  `splunk-system-user`, `sidecar_agent-manager`, and `nobody`, each with a
  human-readable `reason` column. When a future Splunk release introduces
  another internal identity, add a row to the lookup — no code change
  needed.
- Consumed by `mcp_rest_clients` (REST detection), all three discovery
  panels in **MCP Access & Tools**, and Getting Started Step 1.
- Discovered while running live MCP traffic against the official server:
  every user with the `admin` role inherits `mcp_tool_execute` (it surfaced
  as a "full-tool MCP" in the access matrix), and the official server's
  own credential-unlock / housekeeping calls run as `splunk-system-user`
  carrying the same `Splunk_MCP_Server/*` user-agent as real traffic — on
  a 24h trace that was 110 phantom calls next to 99 real ones.

### Changed

#### "Risky queries %" KPI replaces unbounded "Risk score" sum
- The legacy KPI summed every query's risk score (could climb past 1800 —
  meaningless ceiling). The new metric is the **share of MCP queries at
  the MEDIUM band or higher (`risk_score ≥ 3`)**, bounded 0–100, lower
  is better. Shows up on both **MCP Overview** and **Quality & Hygiene**.

#### `is_no_time_bound` weight set to 0 (was 2)
- MCP servers pass the search time range as an API parameter (the
  dispatched `earliest`/`latest` arrive at the search head out-of-band),
  not inside the SPL text the macro inspects. The flag therefore fires on
  virtually every MCP query — leaving it at the v1.0 weight of +2 would
  push almost every MCP query into the LOW band, drown out real
  anti-patterns, and force a downstream "ignore LOW" workaround in every
  consumer.
- The flag is still **detected and reported** in dashboards (transparency
  — users can audit the behaviour) but does not contribute to
  `risk_score`. Override the weight in `local/macros.conf` if you need
  it counted for human-SPL traffic.

#### MCP Overview top row recomposed
- The standalone Online/Offline status badge is gone; **MCP liveness**
  is now the primary status row, followed by 5 KPIs in a single line
  (Last activity · Queries 24h · Active users · Risky queries % ·
  Unique SPL bodies).

#### REST tool inference (`mcp_rest_tool_infer`) covers more endpoints
- Live MCP traffic on a working homepc install showed ~85% of REST calls
  falling into `other`. The macro now classifies them into actionable
  buckets:
  - SAIA tools (`saia_generate_spl`, `saia_explain_spl`,
    `saia_optimize_spl`, `saia_ask_question`, plus a `saia_other`
    catch-all for SAIA settings reads)
  - `validate_spl` (`/services/search/parser`, fired on every run_query)
  - `search_summary` (`/admin/summarization`, the result-metadata
    preflight)
  - `auth_check` (`/services/authentication/current-context`, the per-call
    auth probe — high volume, useful liveness signal but should not
    inflate "tool usage" panels)
- After the change only generic `/services/apps/local` listings still
  land in `other` on a representative trace. No alert or risk-score
  logic changed; this only refines the labels in the MCP Access and
  Detection dashboards.

### Fixed
- **`admin` no longer appears in MCP Access panels as a capability-granted
  MCP.** Filtered via `mcp_excluded_users` in all three discovery panels
  in `mcp_access.xml`. See the *Reserved-user exclusion* subsection above
  for context.
- **`splunk-system-user` no longer appears as a phantom MCP agent.**
  `mcp_rest_clients` filters against `mcp_excluded_users` instead of
  inline `user!=…` checks, so the same exclusion list covers REST
  detection and dashboards.
- **Getting Started Step 1 no longer lists Splunk's internal identities.**
  The "Who is your MCP agent?" discovery panel had no filter, so
  `splunk-system-user` (~1180 audit searches from scheduled jobs,
  including MCP-Watch's own cron reports) and `sidecar_agent-manager`
  sat above the real MCP account. A first-time operator could plausibly
  add one of those to `mcp_users.csv` and end up labelling every Splunk
  scheduled search as MCP traffic. The panel now filters against
  `mcp_excluded_users`.
- **`mcp_rest_clients` composition documented.** The base-searches
  comment block promised callers could append `earliest=...` to any
  base macro, but `mcp_rest_clients` ends in `| lookup … | where`
  (transforming), so an appended `earliest=` lands past the `where` and
  Splunk rejects it ("Error in 'where' command: The operator at
  'earliest=-1h' is invalid"). The macro's docstring now states that
  this one requires time bounds as dispatch parameters, not appended.
- **MCP Detection dashboard text aligned with the exclusion design.**
  The "Why this dashboard exists" panel and the dashboard description
  used to mention catching MCPs that authenticate as a "shared account
  (e.g. admin)" — that framing now contradicts the exclusion lookup.
  Rewrites the three places (panel HTML, `<description>` tag, README
  bullet) to drop the admin example, mention the exclusion lookup, and
  tell operators how to opt back in if their deployment really does use
  `admin` as the MCP service account.
- **Heartbeat scheduled search no longer logs HTTP 404 every 5 minutes
  when the official Splunk MCP Server isn't installed.** The original
  SPL hit `/services/apps/local/Splunk_MCP_Server` directly; on installs
  without the official server, splunkd logged two ERROR + one WARN line
  per dispatch (12×/hr, forever). Now lists all apps via
  `/services/apps/local` and filters with `where title="..."` — silent
  when not present.
- **Getting Started Step 1 title explains the empty state.** On a
  fresh Splunk where admin and `splunk-system-user` are the only
  accounts that have run searches (both filtered by
  `mcp_excluded_users`), Step 1 returns zero rows. The expanded title
  now tells a first-time operator that empty here means "no real /
  MCP traffic yet — proceed to Step 2".
- **Step 2's bulk-edit SPL drops the default placeholder row.** Previous
  version appended `YOUR_MCP_USER` next to the
  `mcp_service_account` placeholder, leaving both in
  `mcp_users.csv`. Adds `| where user!="mcp_service_account"` at the
  head of the example so the lookup ends up with only the real user.
- Heartbeat dashboard cell now XML-escapes `<` in the freshness query
  (was breaking dashboard parse).
- Heartbeat scheduled search cron is `*/5 * * * *` paired with a 6-min
  freshness threshold (was an alert-style cron — triggered an AppInspect
  "gratuitous cron" warning, now clean).
- User × Tool matrix no longer errors when no users have the
  `mcp_tool_execute` capability.

### AppInspect
- `--mode precert`: 0 failures, 2 informational warnings
  (`check_for_updates=true` is *required* for Splunkbase apps —
  AppInspect's "private app" hint is a false positive in this context;
  `collections.conf exists` is purely informational, no action required).
- `--included-tags cloud`: 0 failures, 1 informational warning
  (collections.conf KV-store notice).

### Development phases (internal milestones — informational)
- The work shipped here was developed in two phases, tagged `v1.1` and
  `v1.2` internally in git. The phasing is preserved in commit history
  for future archaeology; the Splunkbase release rolls both into a
  single 1.1.0 update.

---

## [1.0.0] — 2026-05-22

Initial Splunkbase release (approved & published 2026-05-27).

### Added
- **Dashboards (3):** MCP Overview · Activity Timeline · Quality & Hygiene.
- **Weighted Risk Score** (anti-pattern weights + off-hours / large-result
  bonuses) and categorical `risk_band` (NONE / LOW / MEDIUM / HIGH /
  CRITICAL).
- **Five anti-patterns** detected via `mcp_antipattern_check` macro:
  `is_wildcard_index`, `is_dbinspect_all`, `is_overly_wide_time`,
  `is_no_time_bound`, `is_len_raw`.
- **Self-Test saved search** validating the regex against
  `lookups/regex_fixtures.csv`.
- **Reports (4) + Alerts (2)** — Daily Query Volume, Anti-Pattern Offenders,
  REST Endpoint Distribution, Top SPLs · Wildcard Index Used,
  Overly Wide Time Range.
- **Lookups:** `mcp_users.csv`, `mcp_tool_catalog.csv`,
  `mcp_tool_denied.csv`, `regex_fixtures.csv`.
- **Macros:** `audit_index`, `internal_index`, `mcp_audit_searches`,
  `mcp_rest_calls`, `mcp_spl_extract`, `mcp_rest_path`,
  `mcp_antipattern_check`, `mcp_risk_score`.
- **Eventtypes:** `is_mcp_query`, `is_anti_pattern_query`,
  `is_high_risk_query`.
- AppInspect clean: `--mode precert` + `--included-tags cloud`
  → 0 failures (1 informational KV-store notice).

---

[1.1.0]: https://github.com/ALPERKESKE/mcp-watch/releases/tag/v1.1.0
[1.0.0]: https://github.com/ALPERKESKE/mcp-watch/releases/tag/v1.0.0
