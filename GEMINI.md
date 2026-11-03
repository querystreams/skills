# Working with Query Streams

Query Streams gives you live, read-only access to this organization's databases through its Network Agents.
Every tool call is logged to the account, and every reply counts against the organization's data allowance,
so work the way a careful analyst would: learn the schema first, ask small, and reuse results you already have.

## 1. Before you write any SQL

Follow this order on every new database, every session — even when you believe you already know the product
or the schema. This organization's schema is the truth; your prior knowledge of a vendor's data model is not.

1. `qs_list_connectors` — the connectors and databases you may query, each with its `connection_state` and
   `si_state`. Every database listed is queryable, whatever its capture state.
2. `qs_get_connector_schema` — the tables of one database. `detail=domains` groups them by business area;
   it does **not** return columns.
3. `qs_get_table_schema` — once for **each table** you are about to put in SQL. It returns the exact column
   names, types, keys, allowed values and, where Semantic Intelligence has run, a description of what each
   column means. This is the step that gets skipped, and skipping it is where almost every failed query comes from.
4. `qs_run_query` — the SELECT.

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
| SQL Server | `SELECT TOP n …` (after `DISTINCT` if present); `OFFSET … FETCH` needs `ORDER BY` | `[name]` | no `LIMIT`; `GETDATE()`; string concat `+` |
| PostgreSQL | `LIMIT n` | `"name"` (case-sensitive when quoted) | `::type` casts; `NOW()`; `ILIKE` |
| MySQL / MariaDB | `LIMIT n` | `` `name` `` | `NOW()`; `CONCAT()`; backticks not double quotes |
| Oracle | `FETCH FIRST n ROWS ONLY` | `"NAME"` (upper-case unless quoted at creation) | no `LIMIT`/`TOP`; `SYSDATE`; `FROM DUAL` for scalar selects |
| Snowflake | `LIMIT n` | `"NAME"` | upper-case unquoted names; `CURRENT_TIMESTAMP()` |
| BigQuery | `LIMIT n` | `` `project.dataset.table` `` | fully qualified tables; `SAFE_CAST`; partition filters may be required |
| SQLite / DuckDB | `LIMIT n` | `"name"` | DuckDB: `::type`; SQLite: loose typing |
| Microsoft Access | `SELECT TOP n …` | `[name]` | `IIf()` not `CASE`; `#date#` literals; `&` concat |

## 3. Reading a result

- `is_truncated` true and `truncated_by` tell you rows were left out and why. Say so to the user; do not
  present a partial result as complete.
- `data_source` on every reply names the tier that answered (`schema_intelligence`, `captured_schema`,
  `live_schema`). Mention it only when it changes what the user should trust.
- Prefer aggregates in SQL over pulling rows and summing them yourself; it is faster, exact, and costs less allowance.

## 4. When the result is big, or the user will ask follow-ups

On plans with **Result analysis**, add `hold=true` to `qs_run_query` or `qs_run_saved_query`. The Network Agent
keeps the complete result ready for a limited time, and every follow-up question is answered from that copy
with `qs_result` — never by running the database query again:

- `qs_result op=query` — a SELECT over the held result (filter, group, join two held results).
- `op=page` / `op=sample` — browse or sample it.
- `op=windows` lists what is held; `qs_result_release` frees one when you are done.
- `group_by` + `aggregate` with `hold=true` returns totals in the same call as the hold.

On **Deeper analysis** plans, `qs_result op=anomalies | segments | compare | diff` analyse a held result or
two held results. Reach for them before writing your own statistics.

A follow-up that comes back `MCP_RESULT_EXPIRED` means the result is no longer held; run the query again with `hold=true`.

## 5. Understanding a table before querying it

- `qs_profile_table` — distributions, null rates, distinct counts and sample values for one table (stored answer
  with the Read scope; a fresh profile needs Analyze and spends the organization's credits — ask before forcing it).
- `qs_get_relationships` — the joins between tables, formal keys first, then discovered ones with a confidence.
  Use it instead of guessing join columns.
- `qs_data_freshness` — the newest timestamp in a table; pass it back as `since_value` on `qs_run_query` to read
  only what changed.
- `qs_get_si_status` — whether Semantic Intelligence has run on a database and how complete it is.

## 6. Repeating questions

- `qs_list_saved_queries` + `qs_run_saved_query` — the organization's saved and federated queries, parameters
  included. A question that comes back weekly belongs here, not in ad-hoc SQL.
- `qs_alerts op=rules | state | recent` reads alert rules and their state; `qs_create_alert_rule` proposes one
  (always a dry run — a person confirms it in Query Streams).

## 7. When something goes wrong

| You see | It means | Do |
|---|---|---|
| `MCP_AGENT_QUERY_ERROR … class=unknown_column` | a column you named does not exist | use the column list in the message; re-read the table schema if it is not there |
| `MCP_AGENT_QUERY_ERROR … class=syntax` | the SQL is not valid for this engine | check the dialect table above |
| `MCP_AGENT_NOT_CONNECTED` / `unstable` | the Network Agent is offline or reconnecting | tell the user; do not retry in a loop |
| `MCP_QUERY_TIMEOUT` | the query ran past its time limit | narrow it (WHERE, fewer columns) or size it with `mode=shape` |
| `MCP_FEATURE_NOT_INCLUDED` | the plan does not include that capability | continue without it and tell the user once what it would add |
| `MCP_DATA_REALM_EXHAUSTED` / `MCP_USER_BYTES_CAP` / `MCP_USER_CALL_CAP` | the allowance or a cap is used up | stop and tell the user once; the message says how to raise it |
| `MCP_RATE_LIMITED` | too many calls this minute | wait for `retry_after`, then continue |
| `MCP_OUTPUT_TOO_LARGE` | one reply would exceed the plan's size cap | fewer columns or rows, or `hold=true` and query the held result |

Never retry an identical failing query. Every refusal already tells you what to change.

## 8. The data allowance

Every reply — rows, schema reads and held-result pages — counts against the organization's data allowance.
Keep exploratory questions small, prefer one aggregate over many row pulls, and answer follow-ups from a held
result instead of re-running. When a reply carries a `capacity_low` note, pass it on to the user once.

## 9. Do not

- Guess a column or table name — read the table schema.
- Pull a whole table to count or sum it — aggregate in SQL.
- Retry the same failing statement.
- Loop on an offline agent or an exhausted allowance.
- Present a truncated or sampled result as exact.

<!-- querystreams-playbook v2026.11.02 -->
