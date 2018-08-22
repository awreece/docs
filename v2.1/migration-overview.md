---
title: Migration Overview
summary: Learn how to migrate data into a CockroachDB cluster.
redirect_from: import-data.html
toc: true
---

CockroachDB supports importing data from the following formats/applications:

- [CSV/TSV][csv]
- [Postgres][postgres]
- [MySQL][mysql]
- [Oracle][oracle]

{{site.data.alerts.callout_info}}
To import/restore data from CockroachDB-generated [enterprise license backups](backup.html), see [`RESTORE`](restore.html).
{{site.data.alerts.end}}

## See also

- [Migrate from MySQL][mysql]
- [Migrate from Postgres][postgres]
- [Migrate from Oracle][oracle]
- [Migrate from CSV][csv]
- [`IMPORT`](import.html)
- [SQL Dump (Export)](sql-dump.html)
- [Back up Data](back-up-data.html)
- [Restore Data](restore-data.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)

<!-- Links -->

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[oracle]: migrate-from-oracle.html
[csv]: migrate-from-csv.html
