---
title: Restore Data
summary: Learn how to back up and restore a CockroachDB cluster.
toc: false
---

How you restore your cluster's data depends on the type of [backup](back-up-data.html) originally:

Backup Type | Restore using...
------------|-----------------
[`cockroach dump`](sql-dump.html) | [Import data](import-data.html)
[`BACKUP`](backup.html)<br/>(*[enterprise license](https://www.cockroachlabs.com/pricing/) only*) | [`RESTORE`](restore.html)

If you want to import data from another database such as Postgres, MySQL, or Oracle into CockroachDB, see the [Migration Overview](migration-overview.html).

## See also

- [Back up Data](back-up-data.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)
