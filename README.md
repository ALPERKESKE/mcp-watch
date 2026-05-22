# MCP-Watch for Splunk

**Zero-dependency visibility & governance for AI agents (MCP servers) operating against Splunk.**

When an AI agent (Claude, Cursor, …) talks to Splunk over the Model Context Protocol, it leaves a rich trail in `_audit` and `_internal` — but no native dashboard surfaces it. MCP-Watch turns those traces into an operations- and governance-ready view: which queries each agent runs, how often, how risky, against which indexes, and which MCP tools each user can reach.

- **Version:** 1.0.0 · **License:** Apache 2.0
- **Dependencies:** none — no CIM, no Add-on Builder, no companion apps. Reads built-in `_audit` / `_internal` in place.
- **Compatibility:** Splunk Enterprise & Cloud 9.x / 10.x.
- **No restart required** — ships only search-time knowledge objects (no index-time config, inputs, or binaries).
- **AppInspect:** passes `--mode precert` and `--included-tags cloud` (0 failures / 0 warnings).

---

## Install

1. **Install the app** — Splunk Web → *Apps → Install app from file*, or extract to `$SPLUNK_HOME/etc/apps/mcp_watch`. No restart needed.
2. **Tell it who your agents are** — edit `lookups/mcp_users.csv` and list the Splunk username(s) your MCP server authenticates as:
   ```csv
   user,role,description
   claude_mcp,mcp_agent,Primary Claude MCP service account
   cursor_mcp,mcp_agent,Cursor IDE MCP account
   ```
   Anything listed here is treated as agent traffic across every dashboard, report, and alert. Admins can edit it in *Settings → Lookups → Lookup table files*.
3. **(Splunk Cloud only)** if your audit/internal indexes are remapped, override the `audit_index` / `internal_index` macros in `local/macros.conf`.
4. Wait ~5 minutes for the first scheduled searches to populate, then open **Apps → MCP-Watch → MCP Overview**.

> **Confirm attribution:** `index=_audit action=search info=granted earliest=-1h | stats count by user` — the user(s) owning your MCP traffic should appear here and match `mcp_users.csv`.

---

## Dashboards & panels

### 1 · MCP Overview — at-a-glance picture (last 24h)
| Panel | What it shows |
|-------|---------------|
| **Splunk MCP Server — Status** | Green **✓ ONLINE** / red **✗ OFFLINE** badge — whether the official Splunk MCP Server app is installed & enabled. |
| **Last MCP activity** | Minutes since the last call to `/services/mcp` (green < 15m, amber, red). |
| **Queries (last 24h)** | Total SPL queries run by MCP users. |
| **Active MCP users** | Distinct MCP agents seen. |
| **Risk score (24h)** | Sum of every query's risk score (see *Risk scoring*). |
| **Unique SPL bodies** | Distinct queries (deduplication signal). |
| **Query volume — 15-min buckets** | Timechart of query rate. |
| **Top 5 SPL bodies (24h)** | The most frequently run agent queries. |

### 2 · Activity Timeline — what happened, when
| Panel | What it shows |
|-------|---------------|
| **Queries per hour** | Hourly query volume per user. |
| **Latest 50 queries** | Most recent agent SPL with user + time. |
| **REST endpoint distribution (24h)** | Which `splunkd` REST endpoints the agents hit (maps to tool semantics). |
| **REST status code mix** | 2xx/4xx/5xx breakdown of MCP REST calls. |

### 3 · Quality & Hygiene — does the agent write good SPL? (7d)
| Panel | What it shows |
|-------|---------------|
| **Risk score (7d)** | Total weighted risk. The **ⓘ** by the title explains the formula on hover. |
| **Queries with at least one hit** | Count of queries that tripped any anti-pattern. |
| **Worst offender (user)** | The agent with the highest cumulative risk. |
| **Highest risk band (7d)** | Worst single-query band reached (LOW…CRITICAL). |
| **Anti-pattern breakdown** | Hit counts per anti-pattern (wildcard index, `len(_raw)`, no time bound, …). |
| **Hits by user** | Anti-pattern hits grouped by agent. |
| **Risk band distribution (7d)** | How many queries fell into NONE/LOW/MEDIUM/HIGH/CRITICAL. |
| **Off-hours risk events (7d)** | Risky queries run before 07:00 / after 19:00. |
| **Top offending queries** | The actual SPL bodies driving the score, with band + user. |

### 4 · MCP Access & Tools — who can do what
| Panel | What it shows |
|-------|---------------|
| **Users who can call MCP tools** | Accounts whose role grants `mcp_tool_execute`, with their roles. |
| **Access model** | Short note explaining how tool access works here (see below). |
| **User × Tool access matrix** | Matrix of every MCP user × every tool — **✓ granted** (green) / **✗ denied** (red). Denials come from the `mcp_tool_denied.csv` policy lookup. |
| **Tool usage by user (24h)** | Stacked chart of which tools each agent actually used. |
| **Searchable index scope per MCP role** | The real per-user data boundary — which indexes each MCP role may search. |

---

## Risk scoring

Each query gets a **risk score** = weighted sum of detected anti-patterns, plus situational bonuses:

| Signal | Weight |
|--------|:------:|
| wildcard index (`index=*`) | +5 |
| `dbinspect index=*` | +4 |
| overly-wide time window (≥ 30d) | +3 |
| no time bound (no `earliest`/`latest`) | +2 |
| `len(_raw)` | +1 |
| *bonus:* off-hours (before 07:00 / after 19:00) **and** risky | +2 |
| *bonus:* huge result set (> 100k rows) **and** risky | +5 |

**Risk band (per query):** `CRITICAL ≥ 15` · `HIGH ≥ 8` · `MEDIUM ≥ 3` · `LOW ≥ 1` · `NONE = 0`.

Dashboard "Risk score" panels show the **sum** of all queries' scores in the time range — there is no fixed maximum; lower is better (0 = clean).

---

## Reports (scheduled) & alerts

**Reports** (feed the dashboards):
- `MCP-Watch - Daily Query Volume` — per-user daily query counts (7d).
- `MCP-Watch - Anti-Pattern Offenders` — per-user weighted risk breakdown (7d).
- `MCP-Watch - REST Endpoint Distribution` — top REST endpoints per agent (24h).
- `MCP-Watch - Top SPLs` — most frequent agent queries (24h).

**Alerts:**
- `MCP-Watch - Alert - Wildcard Index Used` — severity 4, fires on `index=*`.
- `MCP-Watch - Alert - Overly Wide Time Range` — severity 3, fires on > ~30d windows.

**Self-test:** `MCP-Watch - Self-Test - Anti-Pattern Regex` (manual) — validates the detection regex against `lookups/regex_fixtures.csv`; all-PASS means the patterns are correct.

---

## Access model (important)

In the official Splunk MCP Server, **tool enablement is global** (`mcp_tools_enabled` KV store, no per-user field). Per-user reality is:
- **Can a user call MCP tools at all?** → the `mcp_tool_execute` capability (granted via role).
- **What data can they reach?** → their own Splunk RBAC (searchable index scope).

The **User × Tool matrix** therefore reflects a **governance policy** you maintain in `lookups/mcp_tool_denied.csv` (rows of `user,tool` you consider off-limits). It is a *visibility/intent* layer — the MCP server v1.x does not enforce per-tool-per-user denial itself. For hard enforcement, restrict the underlying capability the tool needs, or front the MCP server with a policy proxy.

---

## Configuration files (`lookups/`)

| File | Purpose |
|------|---------|
| `mcp_users.csv` | **Required.** The Splunk username(s) treated as MCP agents. |
| `mcp_tool_denied.csv` | Optional governance deny policy (`user,tool`); drives the ✗ cells in the access matrix. |
| `mcp_tool_catalog.csv` | Reference catalog of MCP tool names + categories. |
| `regex_fixtures.csv` | Test fixtures for the anti-pattern self-test. |

---

## Notes & soft dependencies

- The **status badge**, **Last MCP activity**, and the **MCP Access & Tools** dashboard assume the *official Splunk MCP Server* (provenance `MCP:Splunk_MCP_Server:*`, endpoint `/services/mcp`). With a different or absent MCP server these panels degrade gracefully (show offline / empty) — the app never errors.
- The **MCP Access & Tools** dashboard runs `| rest /services/authentication/users` and `/authorization/roles`, so a viewer needs a role with `list_users` / REST access (admin / power / sc_admin). The other three dashboards only need read access to `_audit` and `_internal`.
- **Privacy:** processes audit metadata only. No data leaves your Splunk environment. No telemetry.

---

## Support

Community-supported. Questions, bugs, or feature requests:
- **Email:** alperkeske@gmail.com
- **Issues:** open an issue on the project repository.

## Disclaimer

This is a **personal, independently developed** project, created and maintained by the author in a personal capacity. It is **not affiliated with, sponsored by, endorsed by, or connected to the author's employer or any other organization**, and does not represent the views, work product, or interests of any such party.

The software is provided **"as is"**, without warranty of any kind (see the Apache 2.0 license). Splunk, Splunkbase, and related marks are trademarks of their respective owners; this project is an independent third-party work and is **not affiliated with or endorsed by Splunk LLC / Cisco**. "Splunk" is used only in a referential manner to indicate compatibility.

## License & author

Apache License 2.0 — see `LICENSE`. · alper keske · 2026 · alperkeske@gmail.com
