# SQL dialect reminders by engine

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
