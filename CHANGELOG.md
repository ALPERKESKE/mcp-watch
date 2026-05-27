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

#### MCP liveness heartbeats (KV store)
- New `mcp_heartbeat` KV collection. MCP Overview's top row now shows
  per-MCP `● UP` / `○ STALE` from this collection, **decoupled from
  query activity** — an MCP is UP when a fresh heartbeat (≤ 6 min) exists
  for it; otherwise STALE.
- **Auto-heartbeat for the official Splunk MCP Server**: new scheduled
  report `MCP-Watch - Heartbeat - Official MCP Server` writes a heartbeat
  every 5 min when the official server app is enabled. No setup needed
  for the primary use case.
- Custom / external MCPs send their own heartbeat (the MCP runs outside
  Splunk, so it is the only reliable liveness signal) — see README for
  the `batch_save` example.

#### Splunkbase metadata polish
- App icons added (36×36 + 72×72; `appIcon` / `appIconAlt`).
- Support contact (`alperkeske@gmail.com`) in `app.manifest` and README.
- Disclaimers added to README: personal project (no employer / Splunk
  affiliation) and AI-assisted development disclosure.
- `check_for_updates = true` in `app.conf` (Splunkbase requires it).

### Changed

#### "Risky queries %" KPI replaces unbounded "Risk score" sum
- The legacy KPI summed every query's risk score (could climb past 1800 —
  meaningless ceiling). The new metric is the **share of MCP queries at
  the MEDIUM band or higher (`risk_score ≥ 3`)**, bounded 0–100, lower
  is better. Shows up on both **MCP Overview** and **Quality & Hygiene**.
- MEDIUM+ was chosen because MCP servers pass the time range as an API
  parameter, so the *no time bound* signal (+2 = LOW) fires on almost
  every MCP query; `risk_score > 0` would saturate to ~100% and lose
  all signal. MEDIUM+ isolates real anti-patterns.

#### MCP Overview top row recomposed
- The standalone Online/Offline status badge is gone; **MCP liveness**
  is now the primary status row, followed by 5 KPIs in a single line
  (Last activity · Queries 24h · Active users · Risky queries % ·
  Unique SPL bodies).

### Fixed
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
