# MCP-Watch for Splunk

**Zero-dependency visibility & governance for AI agents (MCP servers) operating against Splunk.**

When an AI agent (Claude, Cursor, …) talks to Splunk over the Model Context Protocol, it leaves a rich trail in `_audit` and `_internal` — but no native dashboard surfaces it. MCP-Watch turns those traces into an operations- and governance-ready view: which queries each agent runs, how often, how risky, against which indexes, and which MCP tools each user can reach.

- **Version:** 1.1.0 · **License:** Apache 2.0
- **What's new since 1.0:**
  - **Multi-signal detection** — also catches **custom / community / non-official MCP servers** (no official provenance, generic user-agent) via REST-side user-agent matching + endpoint-based tool inference. New **MCP Detection (REST)** dashboard + `mcp_user_agents.csv` lookup. Splunk's reserved identities (`admin`, `splunk-system-user`, …) are excluded by default via `mcp_excluded_users.csv`.
  - **Unified Access &amp; Tools** — the access model works with the official `mcp_tool_execute` capability AND, where that's absent, falls back to the connecting account's **RBAC + REST-detected tool usage** (so custom MCPs aren't blank).
  - **Liveness heartbeats** — a dedicated **MCP liveness** panel shows per-MCP up/down, decoupled from query activity. The official server is auto-heartbeated; custom MCPs send their own heartbeat into the `mcp_heartbeat` KV collection.
  - **Getting Started** dashboard — in-app setup guide + "is data flowing?" checks.
- **Dependencies:** none — no CIM, no Add-on Builder, no companion apps. Reads built-in `_audit` / `_internal` in place.
- **Compatibility:** Splunk Enterprise & Cloud 9.x / 10.x.
- **No restart required** — ships only search-time knowledge objects (no index-time config, inputs, or binaries).
- **AppInspect:** passes `--mode precert` and `--included-tags cloud` with **0 failures** (one informational notice that the app uses a KV-store collection — no action required).

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

### 0 · Getting Started — first-run setup
In-app onboarding: how to populate `mcp_users.csv`, the current configuration, and live "is data flowing?" checks. Open this first after install.

### 1 · MCP Overview — at-a-glance picture (last 24h)
| Panel | What it shows |
|-------|---------------|
| **MCP liveness — heartbeats** | *(first row)* Per-MCP `● UP` / `○ STALE` from the `mcp_heartbeat` KV collection — the primary status indicator (see *Liveness & heartbeats*). |
| **Last MCP activity** | Minutes since the last MCP/agent REST call (green < 15m, amber, red). |
| **Queries (last 24h)** | Total SPL queries run by MCP users. |
| **Active MCP users** | Distinct MCP agents seen. |
| **Risky queries % · MEDIUM+ (24h)** | Share of MCP queries that reach the MEDIUM risk band or higher (see *Risk scoring*). Bounded 0–100%. |
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
| **Risky queries % · MEDIUM+ (7d)** | Share of MCP queries at MEDIUM band or higher. The **ⓘ** by the title explains the formula on hover. |
| **Queries with at least one hit** | Count of queries that tripped any anti-pattern. |
| **Worst offender (user)** | The agent with the highest cumulative risk. |
| **Highest risk band (7d)** | Worst single-query band reached (LOW…CRITICAL). |
| **Anti-pattern breakdown** | Hit counts per anti-pattern (wildcard index, `len(_raw)`, no time bound, …). |
| **Hits by user** | Anti-pattern hits grouped by agent. |
| **Risk band distribution (7d)** | How many queries fell into NONE/LOW/MEDIUM/HIGH/CRITICAL. |
| **Off-hours risk events (7d)** | Risky queries run before 07:00 / after 19:00. |
| **Top offending queries** | The actual SPL bodies driving the score, with band + user. |

### 4 · MCP Access & Tools — who can do what *(unified: official + custom)*
Works whether or not the official MCP Server is present. With the official server, it uses the `mcp_tool_execute` capability + tool catalog; otherwise it derives access from the connecting account's RBAC + REST-detected tool usage.
| Panel | What it shows |
|-------|---------------|
| **MCP accounts** | Accounts treated as MCP clients — via the `mcp_tool_execute` capability **or** user-agent detection — with their roles. |
| **User × Tool matrix** | Per account × tool — **✓ granted / ✗ denied** (official, from `mcp_tool_denied.csv`) and **✓ used** (custom, inferred from REST). |
| **Tool usage by account** | Stacked chart of inferred tool usage per account. |
| **Searchable index scope per MCP account** | The real data boundary — which indexes each MCP account's roles may search. |

### 5 · MCP Detection (REST) — *new in 1.1*
Catches MCP/agent clients that v1.0's provenance-only logic missed (custom / community MCP servers, federation gateways).
| Panel | What it shows |
|-------|---------------|
| **MCP / agent clients detected by user-agent** | `splunkd_access` clients whose user-agent matches `mcp_user_agents.csv` — request count, distinct endpoints, users. |
| **REST activity by inferred tool** | Endpoint-based fallback tool attribution (e.g. `/search/jobs` → run_query) when provenance is absent. |
| **Top REST endpoints from MCP / agent clients** | Which REST endpoints non-official clients hit. |
| **Detection signal coverage** | Official-provenance clients vs. those detected by user-agent only. |

> Detection is driven by `lookups/mcp_user_agents.csv` (wildcard user-agent patterns). Tune it for your environment — generic patterns like `python-httpx*` may match non-MCP automation.

---

## Liveness &amp; heartbeats

The **MCP liveness — heartbeats** panel (first row of *MCP Overview*) shows per-MCP status using a fresh-heartbeat check in the `mcp_heartbeat` KV collection, layered with a recent-activity check:

- **`● UP`** — heartbeat ≤ 6 min old **AND** at least one MCP query in the last hour. The MCP server is running and the agent is actively using it.
- **`◐ IDLE`** — heartbeat ≤ 6 min old but **no** MCP queries in the last hour. The MCP server is running but its agent is idle (e.g., Cursor open in the background without an active conversation).
- **`○ STALE`** — no fresh heartbeat. The MCP server itself is down or has been disconnected.

This separation matters for triage: *all* agents simultaneously IDLE often means the AI tool side is broken (Anthropic outage, IDE crashed), while *one* agent STALE while others are UP usually means that specific MCP server stopped.

- **Official Splunk MCP Server** → auto-heartbeated for you: the scheduled search *MCP-Watch - Heartbeat - Official MCP Server* writes a heartbeat every 5 min while the app is enabled. No setup needed.
- **Custom / external MCP** → must send its own heartbeat (it runs outside Splunk, so that is the only reliable liveness signal). Have the MCP, or a sidecar next to it, upsert one row every ~60s — for example:

```bash
curl -sk -u <user>:<pass> \
  "https://<splunk-host>:8089/servicesNS/nobody/mcp_watch/storage/collections/data/mcp_heartbeat/batch_save" \
  -H "Content-Type: application/json" \
  -d "[{\"_key\":\"my-mcp\",\"mcp_id\":\"my-mcp\",\"last_seen\":$(date +%s),\"host\":\"$(hostname)\",\"kind\":\"custom\"}]"
```

Schedule it (cron `* * * * *`, a sidecar, or inside the MCP itself). No heartbeat sender ⇒ that MCP shows `○ STALE` (the official server is the exception — it is auto-heartbeated).

---

## Risk scoring

Each query gets a **risk score** = weighted sum of detected anti-patterns, plus situational bonuses:

| Signal | Weight |
|--------|:------:|
| `\| outputcsv` / `\| outputlookup` / `\| collect` / `\| sendemail` / `\| delete` | **+15** |
| wildcard index (`index=*`) | +5 |
| `dbinspect index=*` | +4 |
| overly-wide time window (≥ 30d) | +3 |
| `len(_raw)` | +1 |
| *bonus:* off-hours (before 07:00 / after 19:00) **and** risky | +2 |
| *bonus:* huge result set (> 100k rows) **and** risky | +5 |

**Risk band (per query):** `CRITICAL ≥ 15` · `HIGH ≥ 8` · `MEDIUM ≥ 3` · `LOW ≥ 1` · `NONE = 0`.

The headline KPI is **Risky queries %** — the share of queries at the **MEDIUM band or higher** (`risk_score ≥ 3`), bounded 0–100%, lower is better.

> **`is_export_or_delete` is the only single-fire CRITICAL.** This flag isn't a "sloppy SPL" pattern — it captures *agent intent* to write data out (`| outputcsv`, `| outputlookup`, `| collect`, `| sendemail`) or destroy data (`| delete`). For an MCP service account these commands are almost never legitimate, so one fire alone tips into CRITICAL and triggers the dedicated alert.

> **`is_no_time_bound` has weight 0** — still detected (and reported in dashboards for transparency), but does not contribute to risk. MCP servers pass the search time range as an API parameter (out-of-band from the SPL text), so the signal would otherwise fire on virtually every MCP query and saturate the score with noise. Override the weight in `local/macros.conf` if you need it counted in a human-SPL context.

---

## Reports (scheduled) & alerts

**Reports** (feed the dashboards):
- `MCP-Watch - Daily Query Volume` — per-user daily query counts (7d).
- `MCP-Watch - Anti-Pattern Offenders` — per-user weighted risk breakdown (7d).
- `MCP-Watch - REST Endpoint Distribution` — top REST endpoints per agent (24h).
- `MCP-Watch - Top SPLs` — most frequent agent queries (24h).
- `MCP-Watch - Heartbeat - Official MCP Server` — auto-heartbeats the official server every 5 min (liveness); writes nothing if that app isn't installed.

**Alerts:**
- `MCP-Watch - Alert - Export or Delete Command Used` — **severity 5 (CRITICAL)**, fires when an MCP user runs `| outputcsv`, `| outputlookup`, `| collect`, `| sendemail`, or `| delete`. Single occurrence is worth waking someone up.
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
| `mcp_user_agents.csv` | Wildcard user-agent patterns used to detect non-official / custom MCP clients on the REST side. |
| `mcp_heartbeat` *(KV)* | Liveness heartbeats — one row per MCP (`mcp_id`, `last_seen`, `host`, `kind`); upserted by the MCP/sidecar. |
| `regex_fixtures.csv` | Test fixtures for the anti-pattern self-test. |

---

## Notes & soft dependencies

- Some signals are richest with the *official Splunk MCP Server* (provenance `MCP:Splunk_MCP_Server:*`, endpoint `/services/mcp`, the `mcp_tool_execute` capability). With a different or absent MCP server, MCP-Watch falls back to user-agent detection, REST-inferred tools, RBAC, and heartbeats — panels degrade gracefully (never error).
- The **MCP Access & Tools** dashboard runs `| rest /services/authentication/users` and `/authorization/roles`, so a viewer needs a role with `list_users` / REST access (admin / power / sc_admin). The other three dashboards only need read access to `_audit` and `_internal`.
- **Privacy:** processes audit metadata only. No data leaves your Splunk environment. No telemetry.

---

## Support

Community-supported. Questions, bugs, or feature requests:
- **Email:** alperkeske@gmail.com
- **Issues:** open an issue on the project repository.

## Disclaimer

This is a **personal, independently developed** project, created and maintained by the author in a personal capacity. It is **not affiliated with, sponsored by, endorsed by, or connected to the author's employer or any other organization**, and does not represent the views, work product, or interests of any such party.

The software is provided **"as is"**, without warranty of any kind (see the Apache 2.0 license). It was **built with AI assistance (Anthropic's Claude)** and may contain errors — bug reports and fixes are welcome via the project's [issues](https://github.com/ALPERKESKE/mcp-watch/issues).

Splunk, Splunkbase, and related marks are trademarks of their respective owners; this project is an independent third-party work and is **not affiliated with or endorsed by Splunk LLC / Cisco**. "Splunk" is used only in a referential manner to indicate compatibility.

## License & author

Apache License 2.0 — see `LICENSE`. · alper keske · 2026 · alperkeske@gmail.com
