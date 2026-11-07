# Working with Query Streams

Query Streams gives you live, read-only access to this organization's databases through its Network Agents.
Every tool call is logged to the account, and every reply counts against the organization's data allowance,
so work the way a careful analyst would: learn the schema first, ask small, and reuse results you already have.

## 1. Before you write any SQL

Follow this order on every new database, every session — even when you believe you already know the product
or the schema. This organization's schema is the truth; your prior knowledge of a vendor's data model is not.

1. `qs_list_connectors` — the connectors and databases you may query, each with its `connection_state` and
   `si_state`. Every database listed is queryable, whatever its capture state. An API connector (Stripe,
   Shopify, HubSpot, YouTube…) whose database carries `sync_state: initial_sync_running` or
   `waiting_on_provider` is still filling its tables from the vendor's API: an empty or short result there
   means "not synced yet", not "no data" — say so, and offer to retry once the sync has finished.
2. `qs_get_connector_schema` — the tables of one database (`detail=domains` groups them by business area;
   large databases page). It does **not** return columns. When you know what you need but not the table's
   name — on any database above 40 tables, this is the normal case — call `qs_find_tables` with a few words,
   a question or part of a name (any language): it returns the ranked tables with why each matched and the
   exact `qs_get_table_schema` call for the top hit. Search, do not page.
3. `qs_get_table_schema` — once for **each table** you are about to put in SQL. It returns the exact column
   names, types, keys, allowed values and, where Semantic Intelligence has run, a description of what each
   column means. This is the step that gets skipped, and skipping it is where almost every failed query comes from.
4. `qs_run_query` — the SELECT.

Each database's `si_state` on the connector list tells you how much Semantic Intelligence has run on it.
When a database is marked `schema_intelligence`, the table schema carries curated descriptions, semantic types
and sample values — trust them as given. `captured_schema` is bare structure; `live_schema` means the database
is not captured, so the tool reads names and types off the live database for you. In all three cases the
answer is authoritative for that table; never substitute a column name you remember from elsewhere.

If a query fails with an unknown column or table, the error already contains the table's real columns or the
closest table names. Use them. Do not try a second guess.

## 2. Writing SQL that runs on the first try

- One read-only SELECT per call. The Network Agent rejects anything else (`READONLY_VIOLATION`).
- Quote identifiers the way the engine expects; copy names exactly as the table schema shows them, including case.
- Limit rows at the source. Your plan sets how many rows one call returns; asking for more returns the plan's
  limit with `preview_limit_clamped=true`, never an error.
- Size a large table before reading it: `qs_run_query` with `mode=shape` returns row and byte estimates without
  returning rows.

Dialect reminders by engine (the connector row tells you the engine):

| Engine | Row limit | Identifier quoting | Watch out for |
|---|---|---|---|
| SQL Server | `SELECT TOP n …` (after `DISTINCT` if present); `OFFSET … FETCH` needs `ORDER BY` | `[name]` | no `LIMIT`; `GETDATE()`; string concat `+`; window functions, `PERCENTILE_CONT` and `STRING_AGG` depend on the database's compatibility level — on an older database fall back to `ROW_NUMBER()` / `FOR XML PATH`, or compute them over a held result |
| PostgreSQL | `LIMIT n` | `"name"` (case-sensitive when quoted) | `::type` casts; `NOW()`; `ILIKE` |
| MySQL / MariaDB | `LIMIT n` | `` `name` `` | `NOW()`; `CONCAT()`; backticks not double quotes |
| Oracle | `FETCH FIRST n ROWS ONLY` | `"NAME"` (upper-case unless quoted at creation) | no `LIMIT`/`TOP`; `SYSDATE`; `FROM DUAL` for scalar selects |
| Snowflake | `LIMIT n` | `"NAME"` | upper-case unquoted names; `CURRENT_TIMESTAMP()` |
| BigQuery | `LIMIT n` | `` `project.dataset.table` `` | fully qualified tables; `SAFE_CAST`; partition filters may be required |
| SQLite / DuckDB | `LIMIT n` | `"name"` | DuckDB: `::type`; SQLite: loose typing |
| Microsoft Access | `SELECT TOP n …` | `[name]` | `IIf()` not `CASE`; `#date#` literals; `&` concat |
| File sets & API connectors | `LIMIT n` | `"name"` | each file or API object is a table; standard analytical SQL (CTEs, window functions, `QUALIFY`); read the table schema — field names come from the source, not from the API's docs |

## 3. Reading a result

- `is_truncated` true and `truncated_by` tell you rows were left out and why. Say so to the user; do not
  present a partial result as complete.
- Shape the reply before you ask for it: `columns` returns only the columns you need, `max_tokens` bounds the
  reply to what your context can hold, `max_cell_chars` trims long text cells. Smaller replies cost less of the
  allowance and leave room for the answer. (Plans with **Context management**; on other plans the inputs are
  refused with `MCP_FEATURE_NOT_INCLUDED` — carry on without them.)
- `data_source` on every reply names the tier that answered (`schema_intelligence`, `captured_schema`,
  `live_schema`). Mention it only when it changes what the user should trust.
- Prefer aggregates in SQL over pulling rows and summing them yourself; it is faster, exact, and costs less allowance.

## 4. When the result is big, or the user will ask follow-ups

On plans with **Result analysis**, add `hold=true` to `qs_run_query` or `qs_run_saved_query`. The Network Agent
keeps the **complete** result — every row, not the preview you were shown — in a local analytical engine next to
the data, for a limited time. From then on you have a second database that costs the customer's server nothing:
every follow-up is answered from the held copy with `qs_result`, never by running the source query again, and only
the small answer travels to you (so it barely touches the data allowance).

The working pattern: ask the source database **once**, broadly (`hold=true`, the rows you might need, no
premature aggregation), then slice, rank, pivot and compare locally for the rest of the conversation.

- `qs_result op=query` — full analytical SQL over held results: `GROUP BY`, window functions (`ROW_NUMBER`,
  `LAG`, running totals), CTEs, `QUALIFY`, percentiles, date bucketing, and **joins between two held results**
  — even when they came from different databases or different engines. One read-only SELECT per call; the
  held results are the only tables.
- `op=page` / `op=sample` — browse or sample it when the user wants to see rows.
- `op=windows` lists what is held and for how long; `qs_result_release` frees one you have finished with.
- `group_by` + `aggregate` with `hold=true` returns the totals in the same call as the hold — one round trip
  for "give me the breakdown and keep the detail."

On **Deeper analysis** plans the engine also runs four analyses for you — reach for them before writing your own
statistics: `qs_result op=anomalies` (values and periods that break the pattern), `op=segments` (natural groups
in the data), `op=compare` (two held results side by side, statistically), `op=diff` (what changed between two
runs of the same query).

A follow-up that comes back `MCP_RESULT_EXPIRED` means the result is no longer held; run the source query again
with `hold=true`. `MCP_WINDOW_SQL_REJECTED` means the follow-up was not one read-only SELECT over held results.

## 5. Understanding a table before querying it

- `qs_profile_table` — distributions, null rates, distinct counts and sample values for one table (stored answer
  with the Read scope; a fresh profile needs Analyze. The profiling work is free: it spends no credits and no
  allowance; only the reply's bytes count, like every reply. It still runs on the customer's database, so ask
  before forcing it).
- `qs_run_query` with `mode=profile` profiles the *result* of a query instead of a whole table; `mode=quality`
  returns a data-quality report (nulls, duplicates, outliers) — statistics, not rows. Both belong to plans with
  **Result analysis**.
- `qs_get_relationships` — the joins between tables, formal keys first, then discovered ones with a confidence.
  Use it instead of guessing join columns. `qs_discover_relationships` runs a fresh discovery when none are
  recorded (Analyze scope; the discovery is free, no credits and no allowance, but it does work on the database,
  so ask first).
- `qs_data_freshness` — the newest timestamp in a table; pass it back as `since_value` on `qs_run_query` to read
  only what changed.

### Semantic Intelligence

`qs_get_si_status` tells you whether Semantic Intelligence has run on a database and how complete it is. When a
database answers `captured_schema` (structure only) or `not_captured`, the descriptions, semantic types and
enum values that make your SQL right first time are missing. If the key carries the **Analyze** scope:

- `qs_setup_si` captures the database and queues the analysis; `qs_request_si_analysis` re-runs it for tables
  that changed. Both spend the organization's AI credits and data allowance — tell the user what will run and
  get a yes before calling them.
- `qs_get_task_status` follows the queued task; analysis takes minutes, so answer the current question from
  the live schema and offer to continue once it finishes.
- On an API connector that has not finished its first data sync, both answer `MCP_FIRST_SYNC_PENDING`
  (409): there is nothing to analyse yet. Wait for `sync_state` on the connector list to clear, then call again.

Without Analyze, say once that an administrator can run Semantic Intelligence from Query Streams and continue
with the live schema.

## 6. Repeating questions

- `qs_list_saved_queries` + `qs_run_saved_query` — the organization's saved and federated queries, parameters
  included. A question that comes back weekly belongs here, not in ad-hoc SQL.
- `qs_alerts op=rules | state | recent` reads alert rules and their state; `qs_create_alert_rule` proposes one
  (always a dry run — a person confirms it in Query Streams).
- `qs_list_organizations` and `qs_list_agents` — for keys that span several organizations, or to see which
  Network Agent is online before a long query.

## 7. When something goes wrong

| You see | It means | Do |
|---|---|---|
| `MCP_AGENT_QUERY_ERROR … class=unknown_column` | a column you named does not exist | use the column list in the message; re-read the table schema if it is not there |
| `MCP_AGENT_QUERY_ERROR … class=syntax` | the SQL is not valid for this engine | check the dialect table above |
| `MCP_AGENT_NOT_CONNECTED` / `unstable` | the Network Agent is offline or reconnecting | tell the user; do not retry in a loop |
| `MCP_QUERY_TIMEOUT` | the query ran past its time limit | narrow it (WHERE, fewer columns) or size it with `mode=shape` |
| `MCP_FEATURE_NOT_INCLUDED` | the plan does not include that capability | continue without it and tell the user once what it would add |
| `MCP_SCOPE_REQUIRED` / `MCP_CONNECTOR_NOT_ALLOWED` | the key lacks that permission, or the connector is outside the app's approved set | tell the user which permission or connector to add; never work around it |
| `MCP_DATA_REALM_EXHAUSTED` / `MCP_USER_BYTES_CAP` / `MCP_USER_CALL_CAP` | the allowance or a cap is used up | stop and tell the user once; the message says how to raise it |
| `MCP_RATE_LIMITED` | too many calls this minute | wait for `retry_after`, then continue |
| `MCP_OUTPUT_TOO_LARGE` | one reply would exceed the plan's size cap | fewer columns or rows, or `hold=true` and query the held result |

Never retry an identical failing query. Every refusal already tells you what to change.

## 8. What your key is allowed to do

Every key or connected app carries up to three permissions, chosen by the person who created it. Nothing
writes to a database under any of them — the Network Agent accepts SELECT only.

| Permission | Lets you | Tools |
|---|---|---|
| **Read** (always on) | browse connectors, schemas, relationships, saved queries and alert rules — no row data | the list / schema / `qs_get_*` tools, `qs_find_tables`, `qs_result op=windows`, `qs_alerts` |
| **Run queries** (`execute`) | run SELECTs and saved queries, hold results and query them, read freshness, propose an alert rule | `qs_run_query`, `qs_run_saved_query`, `qs_result`, `qs_data_freshness`, `qs_create_alert_rule` |
| **Analyze** | start work on the organization's databases: Semantic Intelligence runs (these spend credits and allowance), fresh profiles and relationship discovery (free) | `qs_setup_si`, `qs_request_si_analysis`, `qs_get_task_status`, `qs_profile_table` / `qs_discover_relationships` with `force_refresh` |

- `MCP_SCOPE_REQUIRED` means the key lacks the permission for that tool. Do not retry or look for a workaround:
  tell the user which permission it needs and that the key's owner can add it on the Query Streams MCP page
  (a connected app has to be approved again with the extra permission).
- A connected app may also be limited to specific connectors; a connector that is not listed for you is not
  missing, it is out of scope (`MCP_CONNECTOR_NOT_ALLOWED`).
- Analyze is deliberately optional because its tools start work on the organization's databases without a
  person in the loop, and Semantic Intelligence runs spend credits and allowance — that is why this playbook
  says to confirm with the user before calling them.

## 9. The data allowance

Every reply — rows, schema reads and held-result pages — counts against the organization's data allowance.
Keep exploratory questions small, prefer one aggregate over many row pulls, and answer follow-ups from a held
result instead of re-running. When a reply carries a `capacity_low` note, pass it on to the user once.

## 10. Gotchas — facts that defy reasonable assumptions

- A connector's *default* database is often a system one (`master` on SQL Server): always pass the `database`
  you mean; `qs_list_connectors` lists them.
- `qs_get_connector_schema` returns tables, never columns — only `qs_get_table_schema` does, and it is the only
  reliable source of column names and case. Names that look obvious from a vendor's documentation are the ones
  that fail.
- Which schema tool: a table name you already know goes to `qs_get_table_schema`; browsing a small database
  goes to `qs_get_connector_schema`; a need whose table name you do not know goes to `qs_find_tables`. Querying
  the catalogue (INFORMATION_SCHEMA, sys.*, pg_catalog) returns bare structure and costs allowance; the schema
  tools return columns, keys and descriptions in one call.
- `si_state = not_captured` on a database is not an error: the schema tools answer from the live database
  (`live_schema`); `qs_get_si_status` / `qs_get_relationships` answer `MCP_DATABASE_NOT_CAPTURED` there by design.
- The row count you get is the plan's limit, not the table's size; `preview_limit_clamped` tells you it was
  clamped. Use `mode=shape` or an aggregate for the real figure.
- The preview you see is not the held result: with `hold=true` the Network Agent keeps **every** row, so
  `qs_result op=query` can aggregate over the full set even when the reply showed 1,000.
- Held results expire on a timer (minutes, longer on bigger plans) and are released when the Network Agent
  restarts; `MCP_RESULT_EXPIRED` means re-run with `hold=true`, not that anything is broken.
- Error messages already carry the fix: `unknown_column` lists the real columns, a wrong table lists similar
  names, `MCP_RATE_LIMITED` gives `retry_after`. Read them before retrying.
- `qs_create_alert_rule` is always a dry run; `qs_setup_si` and `qs_request_si_analysis` spend the
  organization's credits and allowance — confirm with the user first. A forced `qs_profile_table` or
  `qs_discover_relationships` is free (no credits, no allowance; only the reply's bytes count) but still runs
  on the customer's database, so ask first too.
- Every reply is billed to the organization's data allowance, including schema reads and held-result pages;
  errors are not.

## 11. Do not

- Guess a column or table name — read the table schema.
- Pull a whole table to count or sum it — aggregate in SQL.
- Retry the same failing statement.
- Loop on an offline agent or an exhausted allowance.
- Present a truncated or sampled result as exact.
- Rely on a remembered list of connectors, databases or tables — `qs_list_connectors` and the schema tools are
  the only current source.

<!-- querystreams-playbook v2026.11.06.1 -->
