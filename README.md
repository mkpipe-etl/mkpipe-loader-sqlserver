# mkpipe-loader-sqlserver

SQL Server loader plugin for [MkPipe](https://github.com/mkpipe-etl/mkpipe). Writes Spark DataFrames into SQL Server tables via JDBC.

## Documentation

For more detailed documentation, please visit the [GitHub repository](https://github.com/mkpipe-etl/mkpipe).

## License

This project is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE) file for details.

---

## Connection Configuration

```yaml
connections:
  sqlserver_target:
    variant: sqlserver
    host: localhost
    port: 1433
    database: mydb
    user: myuser
    password: mypassword
```

---

## Table Configuration

```yaml
pipelines:
  - name: pg_to_sqlserver
    source: pg_source
    destination: sqlserver_target
    tables:
      - name: public.events
        target_name: dbo.stg_events
        replication_method: full
        batchsize: 10000
```

---

## Write Strategy

Control how data is written to SQL Server:

```yaml
      - name: public.events
        target_name: dbo.stg_events
        write_strategy: upsert       # append | replace | upsert | merge
        write_key: [id]              # required for upsert/merge
```

| Strategy | SQL Server Behavior |
|---|---|
| `append` | Plain `INSERT` via JDBC (default for incremental) |
| `replace` | Drop and recreate table, then insert (default for full). With `if_exists: append`: truncate + insert (preserves schema/indexes) |
| `upsert` | `MERGE target USING temp ON ... WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...;` |
| `merge` | Same as upsert for SQL Server |

---

## Write Parallelism & Throughput

```yaml
      - name: public.events
        target_name: dbo.stg_events
        replication_method: full
        batchsize: 10000
        write_partitions: 4
```

- **`batchsize`**: rows per JDBC batch insert. SQL Server handles 5,000–20,000 well.
- **`write_partitions`**: reduces concurrent JDBC connections via `coalesce(N)`.

---

## All Table Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | string | required | Source table name |
| `target_name` | string | required | SQL Server destination table name (include schema) |
| `replication_method` | `full` / `incremental` | `full` | Replication strategy |
| `batchsize` | int | `10000` | Rows per JDBC batch insert |
| `write_partitions` | int | — | Coalesce DataFrame to N partitions before writing |
| `write_strategy` | string | — | `append`, `replace`, `upsert`, `merge` |
| `write_key` | list | — | Key columns for upsert/merge (required) |
| `if_exists` | string | — | `replace` (drop+create) or `append` (preserve table, truncate+insert). Inherits from settings |
| `dedup_columns` | list | — | Columns used for `mkpipe_id` hash deduplication |
| `tags` | list | `[]` | Tags for selective pipeline execution |
| `pass_on_error` | bool | `false` | Skip table on error instead of failing |
