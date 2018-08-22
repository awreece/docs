---
title: IMPORT
summary: Import data into your CockroachDB cluster.
toc: true
---

The `IMPORT` [statement](sql-statements.html) imports the following types of data into CockroachDB:

- CSV/TSV
- SQL (execute batches of `INSERT` statements)
- Postgres dump files
- MySQL dump files
- Oracle

{{site.data.alerts.callout_info}}
For more information about how to migrate from other databases, see the [Migration Overview](migration-overview.html).
{{site.data.alerts.end}}

## Required privileges

Only the `root` user can run [`IMPORT`](import.html).

## Synopsis

<div>
  {% include {{ page.version.version }}/sql/diagrams/import.html %}
</div>

{{site.data.alerts.callout_info}}The <code>IMPORT</code> statement cannot be used within a <a href=transactions.html>transaction</a>.{{site.data.alerts.end}}

## Parameters

 Parameter | Description
-----------|-------------
 `table_name` | The name of the table you want to import/create.
 `create_table_file` | The URL of a plain text file containing the [`CREATE TABLE`](create-table.html) statement you want to use (see [this example for syntax](#use-create-table-statement-from-a-file)).
 `table_elem_list` | The table definition you want to use (see [this example for syntax](#use-create-table-statement-from-a-statement)).
 `file_to_import` | The URL of the file you want to import.
 `WITH kv_option` | Control your import's behavior with [these options](#import-options).

### Import file URLs

URLs for the files you want to import must use the following format:

{% include {{ page.version.version }}/misc/external-urls.md %}

### Import options

You can control the [`IMPORT`](import.html) process's behavior using any of the following key-value pairs as a `kv_option`.

<a name="delimiter"></a>

| Key          | Value                                                                                                                                                                                 | Required? | Example                                                                                           |
|--------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-----------+---------------------------------------------------------------------------------------------------|
| `delimiter`  | The unicode character that delimits columns in your rows.                                                                                                                             | No        | To use tab-delimited values: `IMPORT TABLE foo (..) CSV DATA ('file.csv') WITH delimiter = e'\t'` |
| `comment`    | The unicode character that identifies rows to skip.                                                                                                                                   | No        | `IMPORT TABLE foo (..) CSV DATA ('file.csv') WITH comment = '#'`                                  |
| `nullif`     | The string that should be converted to *NULL*.                                                                                                                                        | No        | To use empty columns as *NULL*: `IMPORT TABLE foo (..) CSV DATA ('file.csv') WITH nullif = ''`    |
| `skip`       | The number of rows to be skipped while importing a file.                                                                                                                              | No        | To import CSV files with column headers: `IMPORT ... CSV DATA ('file.csv') WITH skip = '1'`       |
| `decompress` | The decompression codec to be used: `gzip`, `bzip`, `auto`, or `none`.  Default: `auto`, which guesses based on file extension (`.gz`, `.bz`, `.bz2`). `none` disables decompression. | No        | `IMPORT ... CSV DATA ('file.csv') WITH decompress = 'bzip'`                                       |

For examples showing how to use these options, see the [Examples](#examples) section.

## Requirements

### Prerequisites

Before using [`IMPORT`](import.html), you should have:

- The schema of the table you want to import.
- The tabular data you want to import (e.g., CSV), preferably hosted on cloud storage. This location *must* be accessible to all nodes using the same address.

For ease of use, we recommend using cloud storage. However, if that isn't available, we have a [guide on running your own file server](create-a-file-server.html).

### Import targets

Imported tables must not exist and must be created in the [`IMPORT`](import.html) statement. If the table you want to import already exists, you must drop it with [`DROP TABLE`](drop-table.html).

You can only import a single table at a time.

You can specify the target database in the table name in the [`IMPORT`](import.html) statement. If it's not specified there, the active database in the SQL session is used.

### Create table

Your [`IMPORT`](import.html) statement must include a `CREATE TABLE` statement (representing the schema of the data you want to import) using one of the following methods:

- A reference to a file that contains a `CREATE TABLE` statement

- An inline `CREATE TABLE` statement

We also recommend [specifying all secondary indexes you want to use in the `CREATE TABLE` statement](create-table.html#create-a-table-with-secondary-and-inverted-indexes). It is possible to [add secondary indexes later](create-index.html), but it is significantly faster to specify them during import.

### Object dependencies

When importing tables, you must be mindful of the following rules because [`IMPORT`](import.html) only creates single tables which must not already exist:

- Objects that the imported table depends on must already exist
- Objects that depend on the imported table can only be created after the import completes

### Available storage

Each node in the cluster is assigned an equal part of the converted CSV data, and so must have enough temp space to store it. In addition, data is persisted as a normal table, and so there must also be enough space to hold the final, replicated data. The node's first-listed/default [`store`](start-a-node.html#store) directory must have enough available storage to hold its portion of the data.

On [`cockroach start`](start-a-node.html), if you set `--max-disk-temp-storage`, it must also be greater than the portion of the data a node will store in temp space.

### Import file location

You can store the tabular data you want to import using remote cloud storage (Amazon S3, Google Cloud Platform, etc.). Alternatively, you can use an [HTTP server](create-a-file-server.html) accessible from all nodes.

For simplicity's sake, it's **strongly recommended** to use cloud/remote storage for the data you want to import. Local files are supported; however, they must be accessible identically from all nodes in the cluster.  In other words, each node's store must have the file in its `extern` directory, i.e. `/path/to/store/extern/data.sql`.

### Table users and privileges

Imported tables are treated as new tables, so you must [`GRANT`](grant.html) privileges to them.

## Performance

All nodes are used during tabular data conversion into key-value data, which means all nodes' CPU and RAM will be partially consumed by the [`IMPORT`](import.html) task in addition to serving normal traffic.

## Viewing and controlling import jobs

After CockroachDB successfully initiates an import, it registers the import as a job, which you can view with [`SHOW JOBS`](show-jobs.html).

After the import has been initiated, you can control it with [`PAUSE JOB`](pause-job.html), [`RESUME JOB`](resume-job.html), and [`CANCEL JOB`](cancel-job.html).

{{site.data.alerts.callout_danger}}Pausing and then resuming an <code>`IMPORT`</code> job will cause it to restart from the beginning.{{site.data.alerts.end}}

## Examples

### Use `CREATE TABLE` statement from a file

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers
CREATE USING 'azure://acme-co/customer-create-table.sql?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
;
~~~

### Use `CREATE TABLE` statement from a statement

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
;
~~~

### Import a tab-separated file

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.tsv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	delimiter = e'\t'
;
~~~

### Skip commented lines

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	comment = '#'
;
~~~

### Skip first *n* lines

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	skip = '2'
;
~~~

### Use blank characters as `NULL`

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	nullif = ''
;
~~~

### Import a compressed CSV file

CockroachDB chooses the decompression codec based on the filename (the common extensions `.gz` or `.bz2` and `.bz`) and uses the codec to decompress the file during import.

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv.gz?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
;
~~~

Optionally, you can use the `decompress` option to specify the codec to be used for decompressing the file during import:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE customers (
		id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
		name TEXT,
		INDEX name_idx (name)
)
CSV DATA ('azure://acme-co/customer-import-data.csv.gz.latest?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co')
WITH
	decompress = 'gzip'
;
~~~

## Known limitation

`IMPORT` can sometimes fail with a "context canceled" error, or can restart itself many times without ever finishing. If this is happening, it is likely due to a high amount of disk contention. This can be mitigated by setting the `kv.bulk_io_write.max_rate` [cluster setting](cluster-settings.html) to a value below your max disk write speed. For example, to set it to 10MB/s, execute:

{% include copy-clipboard.html %}
~~~ sql
> SET CLUSTER SETTING kv.bulk_io_write.max_rate = '10MB';
~~~

## See also

- [Create a File Server](create-a-file-server.html)
- [Migration Overview](migration-overview.html)
- [Migrate from MySQL][mysql]
- [Migrate from Postgres][postgres]
- [Migrate from Oracle][oracle]
- [Migrate from CSV][csv]

<!-- Links -->

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[oracle]: migrate-from-oracle.html
[csv]: migrate-from-csv.html
