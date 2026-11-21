# TSX Data — ooooOasis

Canadian market data for TSX, TSXV, CSE and CBOE Canada listings, served as a REST API and as an MCP server. The snapshot covers 3,219 securities and 3.49M daily bars, plus SEDAR+ filings, corporate actions, trading halts, index membership, exchange listing snapshots, and IIROC/CIRO short-sale reports. Data is served from a dated snapshot — every response family carries its own as-of date, and staleness is labelled as it is, never dressed up as fresh.

Product page: https://www.oooooasis.com/tsx · Pricing: https://api.oooooasis.com/pricing

## Get an API key

Keys are per service — a TSX Data key (prefix `tsx_live_`) works only on this service.

1. Register at https://api.oooooasis.com/register — choose the TSX Data service and enter your email.
2. Click the verification link we email you (valid 24 hours). Your key is shown **once** on that page — store it immediately.
3. Lost your key? Re-issue it at https://api.oooooasis.com/reissue — after email verification the old key is revoked and a new one is shown once.

The free tier is live: **100 calls/day**, no card required. Free keys expire 90 days after issue (register again to renew). Paid tiers are **not on sale yet** — see "Limits & errors" for exactly what a free key can call, and join the waitlist by emailing tsx-data@oooooasis.com.

Send your key on every call in the `Authorization` header:

```
Authorization: Bearer <your key>
```

Keys in URLs (`?api_key=...`) are rejected with a 400 error.

## REST API

Base URL: `https://api.oooooasis.com/tsx` — read-only, GET only (other methods answer 405).

All list endpoints share one paging rule: `limit` defaults to 100 and must be 1–1000; `offset` defaults to 0 and may be up to 10,000. Asking for more answers 400 — responses are never silently truncated. Date ranges (`from`/`to`) are limited to 366 days. Responses carry the echoed query parameters plus `rows` and the result rows; dataset families with a data-date annotation include `as_of` (the table's latest data date in the snapshot).

| Method | Path | Access | Key parameters | Returns |
|---|---|---|---|---|
| GET | `/health` | Public | — | Service status incl. snapshot time (no key, not metered) |
| GET | `/meta` | Public | — | Schema version, snapshot time, per-table row counts and latest dates (no key, not metered) |
| GET | `/securities` | **Free** | `exchange`, `status`, `q`, `limit`, `offset` | Security list / search (`q` matches ticker prefix or company name) |
| GET | `/securities/{security_id}` | **Free** | — | One security by its numeric id (ids come from `/securities`) |
| GET | `/securities/{security_id}/bars` | **Free** | `from`, `to`, `limit`, `offset` | Daily bars for one security (default: most recent 100 rows) |
| GET | `/bars` | **Free** | `date` or `security_id` (+`from`/`to`), `exchange`, `limit`, `offset` | Bars for one date across securities, or a range for one security. `date` or `security_id` is required — no full-table scans |
| GET | `/filings` | Paid | `security_id`, `doc_type`, `from`, `to`, `limit`, `offset` | SEDAR+ filing index. At least one filter is required; `doc_type` must be one of `aif`, `financial_statements`, `material_change`, `mdna`, `ni43101` |
| GET | `/corporate_actions` | Paid | `security_id`, `action_type`, `from`, `to`, `limit`, `offset` | Corporate actions (dividend, split, consolidation, spinoff, merger, name_change, delisting, other). At least one filter is required |
| GET | `/halts` | Paid | `ticker`, `from`, `to`, `limit`, `offset` | Trading halt / resumption events (CIRO). No parameters: most recent 100 |
| GET | `/short_sales` | Paid | `report_date`, `ticker`, `exchange`, `from`, `to`, `limit`, `offset` | Consolidated short positions (IIROC, semi-monthly). No parameters: most recent 100 |
| GET | `/short_sale_trading` | Paid | `period_start`, `period_end`, `ticker`, `listing_market`, `from`, `to`, `limit`, `offset` | Short-sale trading summary (CIRO, semi-monthly). No parameters: most recent period |
| GET | `/index_membership` | Paid | `index_name`, `as_of`, `ticker`, `limit`, `offset` | Index constituents. No parameters: rows of the most recent `as_of` |
| GET | `/listing_snapshots` | Paid | `exchange`, `source`, `limit`, `offset` | Exchange listing snapshots. No parameters: most recent 100 |

**Free** = callable on the free tier (the securities and bars families). **Paid** = requires a paid tier; paid tiers are not on sale yet, so free keys receive `403 tier_not_entitled` on these today.

### Examples

Service metadata — public, no key:

```bash
curl "https://api.oooooasis.com/tsx/meta"
```

Security search — free tier:

```bash
curl -H "Authorization: Bearer $KEY" \
  "https://api.oooooasis.com/tsx/securities?exchange=TSX&limit=100"
```

One day's bars across an exchange — free tier:

```bash
curl -H "Authorization: Bearer $KEY" \
  "https://api.oooooasis.com/tsx/bars?date=2026-11-16&exchange=TSX&limit=100"
```

## MCP server

URL: `https://mcp.oooooasis.com/tsx` — Model Context Protocol (protocol version 2024-11-05), Streamable HTTP (`POST` to this URL). Every request needs your TSX Data key in the `Authorization: Bearer` header; the key selects this service. The same quota, tier scope and coverage metering as REST apply to tool calls.

Generic client configuration:

```json
{
  "mcpServers": {
    "tsx-data": {
      "url": "https://mcp.oooooasis.com/tsx",
      "headers": { "Authorization": "Bearer <your tsx key>" }
    }
  }
}
```

`tools/list` only shows the tools your key can currently call. 12 tools are registered; while paid tiers are not on sale, only the ones marked **Available now** are listed and callable — the rest appear when paid access opens.

| Tool | Available | Parameters | Purpose |
|---|---|---|---|
| `tsx_list_securities` | **Available now** (free) | `exchange`, `status`, `q`, `limit`, `offset` | List securities. Default 100 rows, max 1,000; truncated responses carry `truncated=true` + `total` |
| `tsx_get_security` | **Available now** (free) | `security_id`* | One security by id |
| `tsx_get_bars` | **Available now** (free) | `security_id`*, `from`, `to`, `history` | EOD bars for one security. Default 252 rows, cap 252; pass `history=full` for full history |
| `tsx_get_bars_by_date` | **Available now** (free) | `date`, `exchange`, `limit` | Bars for one date. Default 100 rows, max 1,000 |
| `tsx_get_meta` | **Available now** (not metered) | — | Service metadata: snapshot time, registry counts |
| `tsx_list_filings` | Paid | `security_id`, `doc_type`, `from`, `to` | Filings (SEDAR+). Default 50 rows, max 200 |
| `tsx_list_corporate_actions` | Paid | `security_id`, `from`, `to` | Corporate actions. Default 50 rows, max 200 |
| `tsx_list_halts` | Paid | `from`, `to`, `ticker` | Halts (CIRO). Default 50 rows, max 341 |
| `tsx_get_short_sales` | Paid | `report_date`, `ticker` | Short sales (IIROC). Default 100 rows, max 5,000 |
| `tsx_get_short_sale_trading` | Paid | `period_start`, `ticker` | Short-sale trading. Default 100 rows, max 3,500 |
| `tsx_get_index_membership` | Paid | `index_name`*, `as_of`* | Index membership, one index per period (60 or 220 rows) |
| `tsx_list_listing_snapshots` | Paid | `exchange` | Listing snapshots. Default 50 rows, max 200 |

## Limits & errors

- **Daily quota (free):** 100 calls/day in total, with per-family sub-limits inside it (securities 50/day, bars 80/day). Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` (epoch seconds; quota resets at midnight US/Eastern).
- **Coverage metering:** distinct (ticker × endpoint-family) pairs you touch are counted over a rolling 30-day window; the free-tier cap is 300 pairs. At 80% you get an `X-Coverage-Warning: 80%` header; over the cap, calls answer `429 coverage_exceeded` (pairs roll out of the window daily — there is no fixed reset day).
- **Paging:** `limit` 1–1000 (default 100), `offset` 0–10,000; over-limit answers 400, never a silent truncation. Date ranges are capped at 366 days.
- **Errors** are always JSON: `{"error": {"code": ..., "message": ...}}`. Codes you may see:
  - `400 key_in_url_rejected` — key sent in the URL; use the `Authorization` header.
  - `400 upstream_error` — bad or missing parameters (the message says which).
  - `401 unauthorized` — missing, invalid, revoked or expired key.
  - `403 tier_not_entitled` — your tier may not call that endpoint family.
  - `404 not_found` — unknown path.
  - `429 quota_exceeded` — daily quota (or a family sub-limit) used up; the body carries `limit`, `used`, `scope` and `resets_at`, and the response carries a `Retry-After` header.
  - `429 coverage_exceeded` — rolling 30-day coverage cap reached; the body carries `limit` and `used_30d`.

## Data notes

- **Snapshot-based.** This service reads a dated database snapshot, not a live feed. The current snapshot was built 2026-11-17 from source data through 2026-11-16; `/tsx/meta` always reports the live snapshot time and each table's latest date — check it instead of assuming.
- **Per-family freshness** (latest data date in the current snapshot): daily bars and the security master run through 2026-11-16; filings through 2026-11-15; corporate actions through 2026-11-13; halts through 2026-11-12. Index membership is weekly (latest as-of 2026-11-15). Short-sale reports are semi-monthly with roughly a two-week source lag (latest report date 2026-10-31). Each affected response carries its `as_of` date.
- **Coverage gaps are stated, not filled.** CBOE Canada listings (106 securities) are served in the security master but have **no bars** — the API does not invent them. Delisted and inactive securities remain in the master with their `status` shown.

## Links & contact

- Product page: https://www.oooooasis.com/tsx
- MCP Yellow Page listing: https://mcpyp.com/mcp/com.oooooasis/tsx-data
- Pricing & paid-tier waitlist: https://api.oooooasis.com/pricing
- Sister service — Factor Data (point-in-time equity factors): https://github.com/ooooOasis-fds/factor-data · https://www.oooooasis.com/fct
- Contact: tsx-data@oooooasis.com
