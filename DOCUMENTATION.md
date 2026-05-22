# MCP-Watch for Splunk — App Documentation

> **App ID:** `mcp_watch` · **Version:** 1.0.0 · **License:** Apache 2.0
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
| `is_no_time_bound` | has no `earliest=` / `latest=` clause in the SPL | **2** | Unbounded search. Pragmatic match — picker-only ad-hoc human searches may false-positive; MCP servers typically inject `earliest=` directly, so the signal is reliable for agent traffic. |
| `is_len_raw` | contains `len(_raw)` | **1** | Forces full `_raw` materialization; a common (mostly human) performance footgun. |

> **v1.1 fix:** `is_no_time_bound` previously matched the literal string `_time` anywhere in the SPL, so a query like `index=_audit | bin _time` falsely cleared the flag. The macro now requires an actual `earliest=` / `latest=` clause. Known regex gaps remain for `is_overly_wide_time` (single-digit weeks ≥ 5w aren't matched; v1.1 TODO) — see `lookups/regex_fixtures.csv` for the documented coverage.

`mcp_antipattern_check` also still emits a flat unweighted `antipattern_score` (sum of the five flags) for v1.0 backwards compatibility. New code should use `mcp_risk_score` (next section).

### 2.4 Risk Score (v1.1)

Pipe `mcp_risk_score` after `mcp_antipattern_check` to upgrade from a flat flag-sum to a weighted, context-aware score:

```
risk_score = (is_wildcard_index*5 + is_dbinspect_all*4 + is_overly_wide_time*3
              + is_no_time_bound*2 + is_len_raw*1)
           + (is_off_hours ? 2 : 0)              # only if risk_weight > 0
           + (result_count > 100k ? 5 : 0)       # only if risk_weight > 0
```

Bonuses fire **only when at least one anti-pattern is already present** — a benign query running off-hours doesn't get a phantom risk bump.

The macro also emits a categorical `risk_band` for dashboards / alerts:

| `risk_band` | Threshold | Typical cause |
|---|---|---|
| `CRITICAL` | `risk_score ≥ 15` | `index=*` + `>30d window` + off-hours, or similar combo |
| `HIGH` | `≥ 8` | `index=*` alone, or two mid-tier patterns together |
| `MEDIUM` | `≥ 3` | One mid-tier pattern (e.g. `no_time_bound` + off-hours) |
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

### 3.2 Lookups

- **`lookups/mcp_users.csv`** — columns `user,role,description`. The single source of truth for "who is an agent". Registered as transform `mcp_users` (`transforms.conf`). Admin-writable per `metadata/default.meta`.
- **`lookups/regex_fixtures.csv`** (v1.1) — columns `test_id,anti_pattern,expected,search,notes`. Drives the **Self-Test** saved search (§3.4) — known-positive and known-negative inputs for every anti-pattern regex, including documented `KNOWN_GAP` rows. Ships in the app so anyone editing the regex can verify they haven't regressed.

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

**Alerts** (real-time-ish, run every 5 minutes over the last 15 minutes, 30-minute suppression, tracked):

| Name | Trigger | Severity |
|---|---|---|
| `MCP-Watch - Alert - Wildcard Index Used` | an MCP user runs a search containing `index=*` | 4 (high) |
| `MCP-Watch - Alert - Overly Wide Time Range` | an MCP user runs a search with `earliest` spanning > ~30 days | 3 (medium) |

Each alert returns `_time, user, spl_body` for the offending query. Hook them to email / Slack / ITSI as needed (none wired by default).

**Self-tests** (not scheduled — run on demand):

| Name | What it does |
|---|---|
| `MCP-Watch - Self-Test - Anti-Pattern Regex` *(v1.1)* | Runs `mcp_antipattern_check` against `lookups/regex_fixtures.csv` and groups outcomes by `status` (PASS / FAIL / ERROR_UNKNOWN_PATTERN) and anti-pattern. All-PASS = regex matches its documented fixture set. Any FAIL = inspect the named `test_ids` in the lookup. Use this after editing the regex in `macros.conf`. |

---

## 4. The dashboards

Navigation (`MCP-Watch` app menu): **MCP Overview** (default) · **Activity Timeline** · **Quality & Hygiene** · **Reports** (auto-lists the four reports above) · **Search**.

### 4.1 MCP Overview (`mcp_overview`)
At-a-glance picture of the last 24 hours. Single base search: `` `mcp_audit_searches` earliest=-24h latest=now | `mcp_spl_extract` ``.
- **Queries (last 24h)** — total query count.
- **Active MCP users** — distinct MCP accounts seen.
- **Anti-pattern hits (24h)** — `sum(antipattern_score)` over the window, colour-coded green/amber/red.
- **Unique SPL bodies** — distinct query texts (a proxy for "how varied is the agent's work").
- **Query volume — 15-min buckets** — stacked column `timechart span=15m count by user`.
- **Top 5 SPL bodies (24h)** — the five most-run query texts. *(These are the agent's actual queries — if you see unfamiliar field names here, that's the monitored content, not the app.)*

### 4.2 Activity Timeline (`activity_timeline`)
Drill into individual queries and the REST calls behind them. Inputs: a time-range picker and an **MCP user** dropdown (populated from `mcp_users.csv`, default = all). Base search: `` `mcp_audit_searches` user="$user_filter$" | `mcp_spl_extract` ``.
- **Queries per hour** — stacked column `timechart span=1h count by user`.
- **Latest 50 queries** — `_time, user, spl_body`, newest first (wrapped, drill-down on cell).
- **REST endpoint distribution (24h)** — pie of `uri_path` counts (`` `mcp_rest_calls` | `mcp_rest_path` | stats count by uri_path | head 15 ``). Roughly maps to MCP tool semantics — `search/jobs`, `.../results`, `data/indexes`, `search/parser`, etc.
- **REST status code mix** — bar of HTTP status codes returned to the agent.

### 4.3 Quality & Hygiene (`quality_hygiene`)
Does the agent write good SPL? Base search (7 days): `` `mcp_audit_searches` earliest=-7d latest=now | `mcp_spl_extract` | `mcp_antipattern_check` ``.
- **Anti-pattern hits (7d)** — `sum(antipattern_score)`.
- **Queries with at least one hit** — count of queries where `antipattern_score > 0`.
- **Worst offender (user)** — the account with the highest cumulative score.
- **Anti-pattern breakdown** — bar chart of hit counts per pattern (`index=*`, `len(_raw)`, `dbinspect index=*`, `no time bound`, `>30d window`).
- **Hits by user** — cumulative score per user.
- **Top offending queries** — the worst `spl_body`s by max score then count (top 20, wrapped).

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

Authoritative roadmap lives in `MCPApp.md §13`. Short form:

- **v1.1 P0 — SHIPPED:** weighted Risk Score (this section), `risk_band`, `is_high_risk_query` eventtype, regex fixtures + Self-Test saved search, `is_no_time_bound` regex bug fix.
- **v1.1 P1–P6 (next):** Data Exfiltration detection (high `result_count` + no aggregation); Sensitive Index lookup (`sensitive_indexes.csv`) + ×2 risk multiplier + Governance & Audit dashboard; `setup.json` Universal Setup + in-app Manage MCP Users dashboard; Failure & Recovery dashboard (`info=failed`); Performance Killers regroup (`is_unbounded_join`, `is_values_star`, etc.); prompt/SPL injection session-scope-drift signal.
- **v1.2:** **Sessionization** (group an agent's queries into logical investigation sessions via `streamstats time_window=5m` — not `transaction`); Performance Impact (CPU-seconds / scan_count from `metrics.log` + `_introspection` — no $ figures); Human vs AI comparative baseline (distribution view, not single-number claims).
- **v2.0:** KV Store migration (only if user list grows beyond ~25 rows); multi-MCP-server typology; webhook actions; manual Anthropic token-cost overlay.

---

*MCP-Watch for Splunk — Apache License 2.0 — alper keske, 2026.*
