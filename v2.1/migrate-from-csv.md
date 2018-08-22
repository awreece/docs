---
title: Migrate from CSV
summary: Learn how to migrate data from CSV files into a CockroachDB cluster.
toc: true
---

This page has instructions for migrating data from CSV files to CockroachDB using [`IMPORT`](import.html).

The examples below use CSV files exported from the [employees data set](https://github.com/datacharmer/test_db) that is also used in the [MySQL docs](https://dev.mysql.com/doc/employee/en/).

{{site.data.alerts.callout_success}}
If you use one of the following databases you can avoid CSV altogether and import your data directly:

- [MySQL][mysql]
- [Postgres][postgres]
- [Oracle][oracle]
{{site.data.alerts.end}}

## Step 1. Export data to CSV

Please refer to the documentation of your database for instructions on exporting data to CSV.

You will need to export one (1) CSV file per table in the database.

Note that the CSV files must meet the following requirements:

- Files must be valid [CSV files](https://tools.ietf.org/html/rfc4180), with the caveat that the delimiter must be a single character.  To use a character other than comma (such as a tabs), set a [custom delimiter](import.html#delimiter).
- Files must be UTF-8 encoded.
- If one of the following characters appears in a field, the field must be enclosed by double quotes:
    - delimiter (`,` by default)
    - double quote (`"`)
    - newline (`\n`)
    - carriage return (`\r`)
- If double quotes are used to enclose fields, then a double quote appearing inside a field must be escaped by preceding it with another double quote.  For example:
  `"aaa","b""bb","ccc"`

In addition, there are CockroachDB-specific requirements:

- If a column is of type [`BYTES`](bytes.html), it can either be a valid UTF-8 string or a [hex-encoded byte literal](sql-constants.html#hexadecimal-encoded-byte-array-literals) beginning with `\x`. For example, a field whose value should be the bytes `1`, `2` would be written as `\x0102`.

## Step 2. Host the files where the cluster can access them

Each node in the CockroachDB cluster needs to have access to the files being imported.  There are several ways for the cluster to access the data; for a complete list of the types of storage [`IMPORT`][import] can pull from, see [Import File URLs][import#import-file-urls].

{{site.data.alerts.callout_success}}
We strongly recommend using cloud storage such as Amazon S3 or Google Cloud to host the files with data you want to import.
{{site.data.alerts.end}}

## Step 3. Import the CSV

You will need to write an [`IMPORT TABLE`](import.html) statement that matches the schema of the table you're importing.

For example, to import the `employees.csv` file into an `employees` table, issue the following statement:

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  ) CSV DATA ('s3://acme-co/employees.csv?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456');
~~~

~~~
+--------------------+-----------+--------------------+--------+---------------+----------------+----------+
|       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes   |
+--------------------+-----------+--------------------+--------+---------------+----------------+----------+
| 376482600074444801 | succeeded |                  1 | 300024 |             0 |              0 | 11534293 |
+--------------------+-----------+--------------------+--------+---------------+----------------+----------+
(1 row)
~~~

Repeat the above steps for each table you want to import.

{{site.data.alerts.callout_info}}
For more information about all of the features and options available when using the `IMPORT` statement see [`IMPORT`][import].
{{site.data.alerts.end}}

## See also

- [Migrate from MySQL][mysql]
- [Migrate from Postgres][postgres]
- [Migrate from Oracle][oracle]
- [`IMPORT`][import]
- [SQL Dump (Export)](sql-dump.html)
- [Back up Data](back-up-data.html)
- [Restore Data](restore-data.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)

<!-- Links -->

[postgres]: migrate-from-postgres.html
[mysql]: migrate-from-mysql.html
[oracle]: migrate-from-oracle.html
[import]: import.html
