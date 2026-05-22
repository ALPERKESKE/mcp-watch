# MCP-Watch for Splunk

**Zero-dependency visibility and governance for AI agents (MCP servers) operating against Splunk.**

When an AI agent like Claude talks to Splunk over MCP, it leaves a rich audit trail in `_audit` and `_internal` — but no native dashboard surfaces it. MCP-Watch turns those traces into an operations- and governance-ready view of agentic activity: which queries, how often, by whom, with what efficiency, against which indexes.

- **Version:** 1.0.0
- **License:** Apache 2.0
- **Compatibility:** Splunk Enterprise 9.x, 10.x, and Splunk Cloud (vetted)
- **Dependencies:** none. No CIM, no Add-on Builder, no companion apps.

---

## What it monitors (v1.0)

1. Every SPL the MCP service account ran (`_audit`, `action=search`, `info=granted`)
2. REST endpoint distribution per MCP tool call (`splunkd_access.log`)
3. Anti-pattern detection — `index=*`, `len(_raw)`, `dbinspect index=*`, no time bound, >30-day windows
4. Daily query volume per MCP user
5. Two alerts (wildcard-index, overly-wide time range)
6. Three dashboards: **Overview**, **Activity Timeline**, **Quality & Hygiene**

v1.1 will add a Governance & Audit dashboard, sensitive-index alerts, and off-hours detection. v1.2 will add Investigation Sessions and Performance Impact.

---

## Install

1. Copy `mcp_watch/` to `$SPLUNK_HOME/etc/apps/` (or upload the tarball via Splunk Web → Apps → Install app from file).
2. Restart Splunk (`splunk restart`), or — if you only dropped in conf changes — reload without a restart: `splunk _internal call /debug/refresh -auth <admin>:<pw>`.
3. Edit `lookups/mcp_users.csv` and list the Splunk username(s) your MCP server(s) authenticate as (default: `alper_mcp`). See **Configuration** below.
4. Wait ~5 minutes for the first scheduled searches to populate, then open **Apps → MCP-Watch → MCP Overview**.

### Splunk Cloud

All index references go through the `audit_index` and `internal_index` macros. If your Cloud admin has remapped audit/internal access, override these macros in a `local/macros.conf` — no app code change required.

---

## Configuration

Point the app at the account(s) your MCP server uses by editing **`lookups/mcp_users.csv`**:

```csv
user,role,description
alper_mcp,mcp_agent,Primary Claude MCP service account
cursor_svc,mcp_agent,Cursor IDE MCP account
```

Any user listed here is treated as agent traffic everywhere in the app (dashboards, reports, alerts). Admins with the `admin` role can edit the lookup in place via *Settings → Lookups → Lookup table files*. A proper in-app Universal Setup page is planned for v1.1.

---

## Resource Footprint

MCP-Watch is designed for low overhead in any Splunk environment:

- **No real-time or streaming searches.** All scheduled searches use time-binned summarization (`bin span=1h` or `span=1d`).
- **Search frequency:** 4 scheduled saved searches in v1.0 — total dispatch < 30 seconds/day on a typical 1k-events/sec instance.
- **Search-head load:** measured <1% average CPU on Splunk Enterprise 9.2 with 50 GB/day ingest.
- **No data duplication.** Reads `_audit` and `_internal` in place — no new index, no replication, no summary indexing required in v1.0.
- **Dashboard performance:** every multi-panel dashboard uses a Base Search pattern — Splunk dispatches the heavy search once and panels post-process from cache.
- **Dispatch artifact cleanup:** honors Splunk's standard TTL (typically 10 minutes); no long-lived artifacts left behind.
- **Cloud-compatible:** no on-prem filesystem dependencies; all index references go through macros.

For high-volume environments (>1 TB/day), v1.2 will introduce optional summary indexing — but the v1.0 baseline already runs comfortably within the search-head and indexer budgets of any standard Splunk Enterprise deployment without it.

---

## Privacy

MCP-Watch processes audit data only. It does not transmit any data outside your Splunk environment. It has no telemetry.

---

## Troubleshooting

**No data on the dashboards.**
1. Confirm the Setup user matches the username your MCP server uses to authenticate. Run:
   ```
   index=_audit action=search info=granted earliest=-1h
   | stats count by user
   ```
   The user that owns your MCP traffic should appear here.
2. Confirm the role running the dashboards has read access to `_audit` and `_internal`.

**"Macro not found" / dashboard shows stale content.**
Reload the app's conf: `splunk _internal call /debug/refresh -auth <admin>:<pw>`, or restart Splunk. If a dashboard still looks wrong **only for one user**, that user has a UI-saved copy under `etc/users/<user>/mcp_watch/...` overriding the app default — delete it via *Settings → User Interface → Views*.

---

## Further reading

`DOCUMENTATION.md` (next to this file) walks through every macro, saved search, alert, and dashboard panel in detail — recommended for admins reviewing what the app actually queries.

---

## License

Apache License 2.0 — see `LICENSE`.

## Author

alper keske · 2026
