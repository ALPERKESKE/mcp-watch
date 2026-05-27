# Changelog

All notable changes to **MCP-Watch for Splunk** are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/), and this project
adheres to [Semantic Versioning](https://semver.org/).

---

## [1.2.0] — 2026-05-27

### Added
- **MCP liveness heartbeats (KV store).** New `mcp_heartbeat` KV collection;
  the **MCP Overview** dashboard's top row now shows per-MCP `● UP` / `○ STALE`
  decoupled from query activity. An MCP is UP when a fresh heartbeat
  (≤ 6 min) exists for it; otherwise STALE.
- **Auto-heartbeat for the official Splunk MCP Server.** New scheduled report
  `MCP-Watch - Heartbeat - Official MCP Server` writes a heartbeat every
  5 min when the official server app is enabled. No setup needed for the
  primary use case; custom MCPs send their own heartbeat (see README).

### Changed
- **"Risky queries %" KPI replaces unbounded "Risk score" sum.** The legacy
  KPI summed every query's risk score (could climb past 1800 — meaningless
  ceiling). The new metric is the **share of MCP queries at the MEDIUM band
  or higher (`risk_score ≥ 3`)**, bounded 0–100, lower is better. MEDIUM+
  was chosen because MCP servers pass the time range as an API parameter,
  so the *no time bound* signal (+2 = LOW) fires on almost every MCP query;
  `risk_score > 0` would saturate to ~100% and lose all signal. Shows up
  on both **MCP Overview** and **Quality & Hygiene**.
- **MCP Overview top row recomposed.** The standalone "Online/Offline"
  status badge is gone; **MCP liveness** is now the primary status row,
  followed by 5 KPIs (Last activity, Queries 24h, Active users,
  Risky queries %, Unique SPL bodies) — one row, no wrap.

### Fixed
- Heartbeat dashboard cell now XML-escapes `<` in the freshness query
  (was breaking dashboard parse).
- Heartbeat scheduled search cron is `*/5 * * * *` paired with a 6 min
  freshness threshold (was an alert-style cron, triggered an AppInspect
  "gratuitous cron" warning — now clean).

---

## [1.1.0] — 2026-05 (development; superseded by 1.2.0)

### Added
- **Multi-signal MCP detection.** v1.0 only saw clients carrying the
  official provenance stamp (`MCP:Splunk_MCP_Server:*`). v1.1 also
  recognises custom / community MCP servers and federation gateways
  via **REST-side user-agent matching** (`mcp_user_agents.csv` lookup)
  and **endpoint-based tool inference**. New **MCP Detection (REST)**
  dashboard.
- **Unified MCP Access & Tools dashboard.** Works whether or not the
  official Splunk MCP Server is present. With the official server,
  uses the `mcp_tool_execute` capability + tool catalog; otherwise
  derives access from the connecting account's RBAC + REST-inferred
  tool usage. Custom MCPs no longer leave the dashboard blank.
- **Getting Started dashboard.** In-app onboarding: how to fill
  `mcp_users.csv`, current configuration view, and live "is data
  flowing?" checks.
- **RBAC-based access view as graceful fallback** when
  `mcp_tool_execute` capability is absent (typical for community
  MCPs that authenticate as `admin`).

### Changed
- **App icons** added (36×36 + 72×72; `appIcon` / `appIconAlt`).
- **Support contact** (`alperkeske@gmail.com`) surfaced in
  `app.manifest` and README.
- **`check_for_updates = true`** in `app.conf` — Splunkbase requires
  it not be disabled.
- **Disclaimers** added to README: personal project (no
  employer/Splunk affiliation), and AI-assisted development.

### Fixed
- **User × Tool matrix** no longer errors when no users have the
  `mcp_tool_execute` capability (empty-safe `where isnotnull(user)`
  guard).
- **Default user-agent patterns** in `mcp_user_agents.csv` made
  conservative to avoid flagging non-MCP automation (e.g. generic
  `python-httpx*` requires manual opt-in).

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

[1.2.0]: https://github.com/ALPERKESKE/mcp-watch/releases/tag/v1.2.0
[1.1.0]: https://github.com/ALPERKESKE/mcp-watch/releases/tag/v1.1.0
[1.0.0]: https://github.com/ALPERKESKE/mcp-watch/releases/tag/v1.0.0
