# MCP-Watch for Splunk — App Documentation

> **App ID:** `mcp_watch` · **Version:** 1.1.0 · **License:** Apache 2.0
> **Compatibility:** Splunk Enterprise 9.x / 10.x and Splunk Cloud · **Dependencies:** none

---

## 1. What this app does

When an AI agent (Claude, Cursor, a custom MCP server, …) talks to Splunk over the **Model Context Protocol (MCP)**, every action it takes — each search it dispatches, each REST endpoint it hits — is already recorded in Splunk's own internal logs. Nothing native surfaces it, though.

**MCP-Watch reads that trail and turns it into an operations- and governance-ready picture of agentic activity:** which queries the agent ran, how often, as which account, against which indexes, and how *well-written* those queries were. It answers the question every Splunk admin and CISO eventually asks: *"What is the AI actually doing inside Splunk?"*

It adds **no ingest pipeline, no agent instrumentation, no paid add-on, no new index**. Every data source it needs is present on every Splunk Enterprise instance out of the box.

---

## 2. What it monitors

### 2.1 Data sources (all built-in)

| Log | Index | Sourcetype | What MCP-Watch uses it for |
|---|---|---|---|
| `audit.log` | `_audit` | `audittrail` | Every SPL the MCP account ran, verbatim, with user / time / result. **Primary source.** |
| `splunkd_access.log` | `_internal` | `splunkd_access` | Every REST API call the MCP server made — search dispatch, job-status polling, results fetch, `data/indexes`, etc. Used for the REST-endpoint and status-code views. |

> v1.1+ will also tap `metrics.log` (search-load impact), `scheduler.log` (if an agent triggers saved searches) and per-search `dispatch/*/search.log` (per-search performance). v1.0 stays on the two sources above.

### 2.2 What counts as "MCP traffic"

MCP-Watch identifies agent activity by **user account**. Your MCP server authenticates to Splunk as a dedicated service account (default: `alper_mcp`); MCP-Watch treats every audit/REST event from any user listed in the `mcp_users.csv` lookup as agent traffic. To watch several agents (one for Claude, one for Cursor, …), add a row per account — every search and dashboard picks them all up automatically.

*(The design also anticipates User-Agent– and source-IP–based identification for environments where a service account isn't practical; v1.0 ships the user-account path.)*

### 2.3 The five anti-patterns it flags

Every captured query is scored against five expensive / sloppy SPL patterns (macro `mcp_antipattern_check`). v1.1 attaches a **weight** to each flag and combines them into a `risk_score` (see §2.4).

| Flag | Fires when the query… | Weight | Why it matters |
|---|---|---:|---|
| `is_wildcard_index` | contains `index=*` (case-insensitive) | **5** | Scans every index — one of the costliest mistakes; almost always unintended. |
| `is_dbinspect_all` | contains `dbinspect index=*` | **4** | Enumerates all buckets across all indexes. |
| `is_overly_wide_time` | has `earliest=` spanning roughly **≥ 30 days** (≥ `-30d`, `-N0w`, `-Nmon`, `-Ny`) | **3** | Huge time windows hammer indexers; usually accidental. |
| `is_no_time_bound` | has no `earliest=` / `latest=` clause in the SPL | **0** | **Detected and reported, but not scored.** MCP servers pass the search time range as an API parameter (out-of-band from the SPL text), so this would fire on virtually every MCP query and saturate the risk score. Override the weight in `local/macros.conf` if you need it counted for human-SPL traffic. |
| `is_len_raw` | contains `len(_raw)` | **1** | Forces full `_raw` materialization; a common (mostly human) performance footgun. |
| `is_export_or_delete` | contains `\| outputcsv`, `\| outputlookup`, `\| collect`, `\| sendemail`, or `\| delete` | **15** | **Agent-intent signal, not an SPL-hygiene one.** Single fire → CRITICAL band on its own. For an MCP service account these commands almost never have a legitimate use: scheduled exports are owned by humans via saved searches, and `\| delete` requires the `delete_by_keyword` capability that an `mcp_agent` role should not have. Pairs with a dedicated severity-5 alert. |

> **v1.1 fix:** `is_no_time_bound` previously matched the literal string `_time` anywhere in the SPL, so a query like `index=_audit | bin _time` falsely cleared the flag. The macro now requires an actual `earliest=` / `latest=` clause. Known regex gaps remain for `is_overly_wide_time` (single-digit weeks ≥ 5w aren't matched; v1.1 TODO) — see `lookups/regex_fixtures.csv` for the documented coverage.

`mcp_antipattern_check` also still emits a flat unweighted `antipattern_score` (sum of the five flags) for v1.0 backwards compatibility. New code should use `mcp_risk_score` (next section).

### 2.4 Risk Score (v1.1)

Pipe `mcp_risk_score` after `mcp_antipattern_check` to upgrade from a flat flag-sum to a weighted, context-aware score:

```
risk_score = (is_export_or_delete*15 + is_wildcard_index*5 + is_dbinspect_all*4
              + is_overly_wide_time*3 + is_no_time_bound*0 + is_len_raw*1)
           + (is_off_hours ? 2 : 0)              # only if risk_weight > 0
           + (result_count > 100k ? 5 : 0)       # only if risk_weight > 0
```

> `is_no_time_bound` is intentionally weighted 0 — it would fire on nearly every MCP query (MCP dispatches `earliest`/`latest` as API parameters, not in the SPL the macro inspects). The flag is still detected and shown on dashboards, so users can audit the underlying behaviour without it inflating the risk score.

Bonuses fire **only when at least one anti-pattern is already present** — a benign query running off-hours doesn't get a phantom risk bump.

The macro also emits a categorical `risk_band` for dashboards / alerts:

| `risk_band` | Threshold | Typical cause |
|---|---|---|
| `CRITICAL` | `risk_score ≥ 15` | `index=*` + `>30d window` + off-hours, or similar combo |
| `HIGH` | `≥ 8` | `index=*` alone, or two mid-tier patterns together |
| `MEDIUM` | `≥ 3` | One mid-tier pattern (e.g. `is_overly_wide_time`, or `is_len_raw` + off-hours bonus) |
| `LOW` | `≥ 1` | A single low-weight flag (`len_raw` only, etc.) |
| `NONE` | `0` | Clean query |

`risk_score` is also re-aliased to `antipattern_score`, so any v1.0-era caller that read `antipattern_score` automatically picks up the weighted value once `mcp_risk_score` is in the pipeline. v1.1 P2 will add a sensitive-index ×2 true multiplier on top.

---

## 3. How it's built (knowledge objects)

```
            mcp_users.csv (lookup)
                   │  who is "the agent"
   ┌───────────────┼────────────────────────────┐
   ▼                                            ▼
mcp_audit_searches  ──▶ mcp_spl_extract ──▶ mcp_antipattern_check
(base: _audit search   (adds spl_body)      (adds is_* flags + score)
 events for MCP users)
   │
mcp_rest_calls      ──▶ mcp_rest_path
(base: splunkd_access  (adds uri_path)
 for MCP users)
   │
   ├─▶ eventtypes:  is_mcp_query, is_anti_pattern_query   (+ tags: mcp_activity, governance)
   ├─▶ saved searches (4 reports + 2 alerts, scheduled, offset cron)
   └─▶ 3 dashboards (Base-Search pattern: dispatch once, panels post-process)
```

### 3.1 Macros

| Macro | Definition (essence) | Purpose |
|---|---|---|
| `audit_index` | `index=_audit` | Cloud-portable indirection — override locally if your Cloud admin remapped audit access. |
| `internal_index` | `index=_internal` | Same, for `_internal`. |
| `mcp_user(1)` | `search user=$user$` | Ad-hoc one-off filter for a single account. |
| `mcp_audit_searches` | `` `audit_index` action=search info=granted [ \| inputlookup mcp_users.csv \| fields user ] `` | **Base generating search** — every completed search by any MCP user. Append `earliest=`/`latest=`/`user=` freely. |
| `mcp_rest_calls` | `` `internal_index` sourcetype=splunkd_access [ \| inputlookup mcp_users.csv \| fields user ] `` | **Base generating search** — every REST call by any MCP user. |
| `mcp_spl_extract` | `eval ts=_time \| rex field=search "^search\s+(?<spl_body>.*)"` | Pipe **after** `mcp_audit_searches` to get `spl_body` (the SPL the agent actually typed, minus the leading `search`). |
| `mcp_rest_path` | `rex field=uri "^(?<uri_path>[^?]+)"` | Pipe **after** `mcp_rest_calls` to get `uri_path` (REST path without the query string). |
| `mcp_antipattern_check` | five `eval is_* = if(match(search, …))` + flat `antipattern_score` (sum of flags) | Annotates the pipeline with the anti-pattern flags and the v1.0-compatible flat score. v1.1: regexes are now case-insensitive and `is_no_time_bound` looks for `earliest=`/`latest=` (not just the string `_time`). |
| `mcp_risk_score` | weights the flags (5/4/3/2/1), adds off-hours +2 / `result_count > 100k` +5 bonuses, emits `risk_score` + `risk_band` (NONE/LOW/MEDIUM/HIGH/CRITICAL); also re-aliases `antipattern_score` to `risk_score`. | **v1.1 P0.** Pipe after `mcp_antipattern_check` for governance-grade scoring. See §2.4. |

> **Design note:** the base macros are *generating searches only* (no transforming commands), so callers can safely add time/user filters; the derived fields (`spl_body`, `uri_path`, `is_*`) come from the `*_extract` / `*_check` macros piped afterwards. (Earlier builds folded `rex` into the base macro, which broke any caller that appended `earliest=` — fixed in this version.)

### 3.2 Lookups and KV collections

- **`lookups/mcp_users.csv`** — columns `user,role,description`. The single source of truth for "who is an agent". Registered as transform `mcp_users` (`transforms.conf`). Admin-writable per `metadata/default.meta`.
- **`lookups/regex_fixtures.csv`** (v1.1) — columns `test_id,anti_pattern,expected,search,notes`. Drives the **Self-Test** saved search (§3.4) — known-positive and known-negative inputs for every anti-pattern regex, including documented `KNOWN_GAP` rows. Ships in the app so anyone editing the regex can verify they haven't regressed.
- **`lookups/mcp_user_agents.csv`** (v1.1) — columns `pattern,description`. Wildcard user-agent patterns used by the REST-side detection path to recognise community / custom MCP servers that don't carry the official provenance stamp. Tune for your environment — generic patterns like `python-httpx*` are *opt-in* (commented out by default) so non-MCP automation isn't accidentally flagged.
- **`lookups/mcp_tool_catalog.csv`** (v1.1) — columns `tool,category`. Static catalog of the 14 known MCP tools (used by the **MCP Access & Tools** dashboard's User × Tool matrix; cross-app KV reads of `mcp_tools_enabled` weren't reliable enough).
- **`lookups/mcp_tool_denied.csv`** (v1.1) — columns `user,tool`. Per-user governance deny policy; drives the ✗ cells in the access matrix. **Visibility/intent layer only** — the official MCP server does not enforce this; see §… *Access model*.
- **`mcp_heartbeat` (KV store)** (v1.2) — `collections.conf` collection with fields `mcp_id, last_seen, host, kind, version`. Holds one row per MCP server (official, custom, federation gateway). Upserted by the auto-heartbeat saved search (for the official server) or by the MCP / sidecar itself (for custom MCPs). Drives the **MCP liveness** panel.

### 3.3 Eventtypes & tags

| Eventtype | Search | Tags |
|---|---|---|
| `is_mcp_query` | `` `mcp_audit_searches` `` | `mcp_activity` |
| `is_anti_pattern_query` | `eventtype=is_mcp_query \| `mcp_antipattern_check` \| where antipattern_score > 0` | `mcp_activity`, `governance` |
| `is_high_risk_query` *(v1.1)* | `eventtype=is_mcp_query \| `mcp_antipattern_check` \| `mcp_risk_score` \| where risk_band IN ("HIGH","CRITICAL")` | `mcp_activity`, `governance`, `risk` |

### 3.4 Saved searches

**Reports** (feed dashboards / ad-hoc; scheduled with offset minutes so they don't pile on the top of the hour):

| Name | Schedule | Window | Output |
|---|---|---|---|
| `MCP-Watch - Daily Query Volume` | `5 0 * * *` (daily 00:05) | last 7 days | count of queries per day per MCP user |
| `MCP-Watch - Anti-Pattern Offenders` | `10 * * * *` (hourly) | last 7 days | per-user count + per-pattern breakdown + `score_sum` + the offending SPL bodies |
| `MCP-Watch - REST Endpoint Distribution` | `15 * * * *` (hourly) | last 24 h | top 30 REST `uri_path`s the agents called |
| `MCP-Watch - Top SPLs` | `20 * * * *` (hourly) | last 24 h | top 20 most-frequent `spl_body`s + last-seen time |
| `MCP-Watch - Heartbeat - Official MCP Server` *(v1.2)* | `*/5 * * * *` (every 5 min) | – | Upserts a row into `mcp_heartbeat` KV for `mcp_id=official` whenever the official Splunk MCP Server app is enabled. Paired with a 6-min freshness threshold in the dashboard (so AppInspect doesn't flag the cron as "gratuitous"). Writes nothing if the official server isn't installed. |

**Alerts** (real-time-ish, run every 5 minutes over the last 15 minutes, 30-minute suppression, tracked):

| Name | Trigger | Severity |
|---|---|---|
| `MCP-Watch - Alert - Export or Delete Command Used` | an MCP user runs a search containing `\| outputcsv`, `\| outputlookup`, `\| collect`, `\| sendemail`, or `\| delete` | **5 (critical)** |
| `MCP-Watch - Alert - Wildcard Index Used` | an MCP user runs a search containing `index=*` | 4 (high) |
| `MCP-Watch - Alert - Overly Wide Time Range` | an MCP user runs a search with `earliest` spanning > ~30 days | 3 (medium) |

Each alert returns `_time, user, spl_body` for the offending query. Hook them to email / Slack / ITSI as needed (none wired by default).

**Self-tests** (not scheduled — run on demand):

| Name | What it does |
|---|---|
| `MCP-Watch - Self-Test - Anti-Pattern Regex` *(v1.1)* | Runs `mcp_antipattern_check` against `lookups/regex_fixtures.csv` and groups outcomes by `status` (PASS / FAIL / ERROR_UNKNOWN_PATTERN) and anti-pattern. All-PASS = regex matches its documented fixture set. Any FAIL = inspect the named `test_ids` in the lookup. Use this after editing the regex in `macros.conf`. |

---

## 4. The dashboards

Navigation (`MCP-Watch` app menu): **Getting Started** · **MCP Overview** (default) · **Activity Timeline** · **Quality & Hygiene** · **MCP Access & Tools** · **MCP Detection (REST)** · **Reports** · **Search**.

### 4.0 Getting Started (`getting_started`) — *new in v1.1*
In-app onboarding for new installs. Shows current configuration (which users are in `mcp_users.csv`, audit/internal macros), live "is data flowing?" checks (count of MCP search events seen in the last 24 h), and inline instructions for filling the lookup. Open this first after install.

### 4.1 MCP Overview (`mcp_overview`)
At-a-glance picture of the last 24 hours. Top row (v1.2): **MCP liveness — heartbeats** (per-MCP `● UP` / `○ STALE` from the `mcp_heartbeat` KV collection, decoupled from query activity, freshness ≤ 6 min). Following row, 5 KPIs in a single line:
- **Last MCP activity** — minutes since the last MCP/agent REST call (green < 15m, amber, red).
- **Queries (last 24h)** — total SPL queries run by MCP users.
- **Active MCP users** — distinct MCP accounts seen.
- **Risky queries % · MEDIUM+ (24h)** *(v1.2)* — share of MCP queries that reach `risk_band` ≥ MEDIUM (`risk_score ≥ 3`). Bounded 0–100, lower is better. Replaces the legacy unbounded "Risk score sum" KPI.
- **Unique SPL bodies** — distinct query texts (deduplication signal).

Below: **Query volume — 15-min buckets** timechart + **Top 5 SPL bodies (24h)**.

### 4.2 Activity Timeline (`activity_timeline`)
Drill into individual queries and the REST calls behind them. Inputs: a time-range picker and an **MCP user** dropdown (populated from `mcp_users.csv`, default = all). Base search: `` `mcp_audit_searches` user="$user_filter$" | `mcp_spl_extract` ``.
- **Queries per hour** — stacked column `timechart span=1h count by user`.
- **Latest 50 queries** — `_time, user, spl_body`, newest first (wrapped, drill-down on cell).
- **REST endpoint distribution (24h)** — pie of `uri_path` counts (`` `mcp_rest_calls` | `mcp_rest_path` | stats count by uri_path | head 15 ``). Roughly maps to MCP tool semantics — `search/jobs`, `.../results`, `data/indexes`, `search/parser`, etc.
- **REST status code mix** — bar of HTTP status codes returned to the agent.

### 4.3 Quality & Hygiene (`quality_hygiene`)
Does the agent write good SPL? Base search (7 days): `` `mcp_audit_searches` earliest=-7d latest=now | `mcp_spl_extract` | `mcp_antipattern_check` | `mcp_risk_score` ``.
- **Risky queries % · MEDIUM+ (7d)** *(v1.2)* — same definition as on MCP Overview, 7-day window. Hover the ⓘ for the formula.
- **Queries with at least one hit** — count of queries where `antipattern_score > 0`.
- **Worst offender (user)** — the account with the highest cumulative risk.
- **Highest risk band (7d)** — worst single-query band reached (LOW…CRITICAL).
- **Anti-pattern breakdown** — bar chart of hit counts per pattern (`index=*`, `len(_raw)`, `dbinspect index=*`, `no time bound`, `>30d window`).
- **Hits by user** — cumulative risk per user.
- **Risk band distribution (7d)** — how many queries fell into NONE / LOW / MEDIUM / HIGH / CRITICAL.
- **Off-hours risk events (7d)** — risky queries run before 07:00 / after 19:00.
- **Top offending queries** — the worst `spl_body`s by `risk_score` then count, with band + user.

### 4.4 MCP Access & Tools (`mcp_access`) — *new in v1.1, unified in v1.1*
Who can do what — **unified for official + custom MCPs**. With the official Splunk MCP Server, uses the `mcp_tool_execute` capability + `mcp_tool_catalog.csv`. Without it (community / federation MCPs), falls back to the connecting account's RBAC + REST-detected tool usage so the dashboard never goes blank.
- **MCP accounts** — accounts treated as MCP clients (via the `mcp_tool_execute` capability **or** user-agent detection), with their roles.
- **User × Tool matrix** — per account × tool. ✓ granted / ✗ denied (official, from `mcp_tool_denied.csv`) and ✓ used (custom, inferred from REST endpoints). `chart limit=0` so all 14 catalog tools fit as columns instead of collapsing into "OTHER".
- **Tool usage by account** — stacked chart of inferred tool usage per account.
- **Searchable index scope per MCP account** — the real data boundary — which indexes each MCP account's roles may search. Driven by `rest /services/authorization/roles`, so the viewer needs `list_users` / REST access (admin / power / sc_admin).

### 4.5 MCP Detection (REST) (`mcp_detection_rest`) — *new in v1.1*
Catches MCP / agent clients that v1.0's provenance-only logic missed (custom / community MCP servers, federation gateways like the Splunk cluster MCP).
- **MCP / agent clients detected by user-agent** — `splunkd_access` clients whose user-agent matches `mcp_user_agents.csv` — request count, distinct endpoints, users.
- **REST activity by inferred tool** — endpoint-based fallback tool attribution (e.g. `/search/jobs` → `run_query`) when provenance is absent.
- **Top REST endpoints from MCP / agent clients** — which REST endpoints non-official clients hit.
- **Detection signal coverage** — official-provenance clients vs. those detected by user-agent only.

---

## 5. Configuration

Point the app at the account(s) your MCP server uses by editing **`lookups/mcp_users.csv`**:

```csv
user,role,description
alper_mcp,mcp_agent,Primary Claude MCP service account
cursor_svc,mcp_agent,Cursor IDE MCP account
```

Any user listed here is treated as agent traffic everywhere in the app. (Admins with the `admin` role can edit this lookup in place — see `metadata/default.meta`.)

> **This deployment:** the classic `setup.xml` wizard is disabled (`default/setup.xml` → `setup.xml.disabled`) and `local/app.conf` sets `is_configured = true`; configuration is the CSV above. A proper Universal Setup page is planned for v1.1.

**Splunk Cloud / restricted audit access:** if your admin remapped `_audit`/`_internal`, override `audit_index` / `internal_index` in a `local/macros.conf` — no app-code change needed.

After editing macros or saved searches, reload them: *Settings → Server controls → Restart Splunk*, or `splunk _internal call /debug/refresh`, or `splunk restart`.

---

## 6. Resource footprint

- **No real-time or streaming searches** (the two alerts are scheduled every 5 min over a 15-min window).
- 4 scheduled reports + 2 scheduled alerts — total dispatch well under a minute a day on a typical instance; measured < 1 % average search-head CPU at ~50 GB/day ingest.
- **Base-Search dashboards** — each multi-panel dashboard dispatches its heavy search once; panels post-process from cache.
- **No data duplication** — reads `_audit` / `_internal` in place; no new index, no summary indexing, no replication. (Optional summary indexing for > 1 TB/day environments is a v1.2 item.)
- **No telemetry**, no data leaves your Splunk environment — it processes audit data only.

---

## 7. Troubleshooting

**Dashboards are empty.**
1. Confirm the account in `mcp_users.csv` matches what your MCP server authenticates as:
   `index=_audit action=search info=granted earliest=-1h | stats count by user` — your agent's account should appear.
2. Confirm the role viewing the dashboards can read `_audit` and `_internal`.

**"Error in 'rex' command: Invalid argument: 'earliest=…'".**
You're on an old build where `mcp_audit_searches` ended with `| rex`. Upgrade to this version (base macros are generating-only; `spl_body` now comes from `| `mcp_spl_extract``).

**"The number of wildcards … do not match" on the Quality & Hygiene breakdown panel.**
Old build — `stats … as "index=*"` used a literal `*` in a rename. Fixed here (the chart renames to plain names and relabels via `eval … case()`).

**"Macro not found" / stale dashboard.**
Reload: `splunk _internal call /debug/refresh` (or `splunk restart`). If a dashboard still shows an old version *only for one user*, that user has a UI-saved copy in `etc/users/<user>/mcp_watch/...` overriding the app default — delete it via *Settings → User Interface → Views*.

**"No data on the REST panels."**
REST visibility depends on `splunkd_access.log` capturing the MCP server's calls under a user in `mcp_users.csv`. If your MCP bridge authenticates differently for REST vs. search, add that account too.

**You edited the anti-pattern regex and want to be sure you didn't break it.**
Open Splunk Web → Search & Reporting → Reports, run `MCP-Watch - Self-Test - Anti-Pattern Regex`. All rows should land in the `PASS` status bucket. Any `FAIL` row lists the offending `test_ids` — find them in `lookups/regex_fixtures.csv` to see the input that mismatched. Add a fixture row whenever you fix a bug, so it can't regress.

---

## 8. Roadmap

Authoritative roadmap lives in `MCPApp.md §13` (dev workspace; not shipped in the app). Short form:

- **v1.0 — SHIPPED (Splunkbase, 2026-05-27).** 3 dashboards (MCP Overview, Activity Timeline, Quality & Hygiene), weighted Risk Score + `risk_band`, anti-pattern detection, 4 reports + 2 alerts, regex fixtures + Self-Test, eventtypes.
- **v1.1 — SHIPPED (in repo).** Multi-signal MCP detection (user-agent + endpoint inference) for custom/community MCPs · new **MCP Detection (REST)** dashboard · unified **MCP Access & Tools** dashboard (works with or without `mcp_tool_execute` capability) · **Getting Started** dashboard for in-app onboarding · RBAC-based access fallback · app icons · Splunkbase metadata (support contact, disclaimers).
- **v1.2 — SHIPPED (in repo, this release).** **MCP liveness heartbeats** (KV `mcp_heartbeat`) decoupled from query activity · **auto-heartbeat saved search** for the official server (every 5 min) · **"Risky queries %" MEDIUM+ KPI** (bounded 0–100; replaces unbounded risk-score sum) · MCP Overview top row recomposed (MCP liveness first row, 5 KPIs in one line, status badge removed).
- **v1.3 (next):** Data Exfiltration detection (high `result_count` + no aggregation); Sensitive Index lookup (`sensitive_indexes.csv`) + ×2 risk multiplier + Governance & Audit dashboard; `setup.json` Universal Setup + in-app Manage MCP Users dashboard; Failure & Recovery dashboard (`info=failed`); Performance Killers regroup (`is_unbounded_join`, `is_values_star`, etc.); prompt/SPL injection session-scope-drift signal.
- **v1.4:** **Sessionization** (group an agent's queries into logical investigation sessions via `streamstats time_window=5m` — not `transaction`); Performance Impact (CPU-seconds / scan_count from `metrics.log` + `_introspection` — no $ figures); Human vs AI comparative baseline (distribution view, not single-number claims).
- **v2.0:** KV Store migration of `mcp_users` (only if user list grows beyond ~25 rows); multi-MCP-server typology; webhook actions; manual Anthropic token-cost overlay.

---

*MCP-Watch for Splunk — Apache License 2.0 — alper keske, 2026.*
