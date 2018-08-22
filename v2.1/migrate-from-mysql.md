---
title: Migrate from MySQL
summary: Learn how to migrate data from MySQL into a CockroachDB cluster.
toc: true
---

This page has instructions for getting data from MySQL dump files into CockroachDB using [`IMPORT`](import.html).

The examples below use the [employees data set](https://github.com/datacharmer/test_db) that is also used in the [MySQL docs](https://dev.mysql.com/doc/employee/en/).

## Using `MYSQLDUMP`

### Step 1. Dump the MySQL database to SQL files

You will need to export one (1) MySQL dump file per table in the database.

To export the `employees` table, run the following command:

{% include copy-clipboard.html %}
~~~ shell
$ mysqldump -uroot employees employees > employees.sql
~~~

## Step 2. Host the files where the cluster can access them

Each node in the CockroachDB cluster needs to have access to the files being imported.  There are several ways for the cluster to access the data; for a complete list of the types of storage [`IMPORT`][import] can pull from, see [Import File URLs][import#import-file-urls].

{{site.data.alerts.callout_success}}
We strongly recommend using cloud storage such as Amazon S3 or Google Cloud to host the files with data you want to import.
{{site.data.alerts.end}}

The file needed for this example is `employees.sql`.

### Step 3. `IMPORT` the SQL files

To import the MySQL dump file for a table, issue an `IMPORT` statement like the one shown below.

You will need to look at the dump file and translate the MySQL [`CREATE TABLE`](create-table.html) statement there into something CockroachDB understands.

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name STRING NOT NULL,
    last_name STRING NOT NULL,
    gender STRING NOT NULL,
    hire_date DATE NOT NULL
  )
  MYSQLDUMP DATA ('s3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456');
~~~

Success will look like:

~~~
+--------------------+-----------+--------------------+------+---------------+----------------+-------+
|       job_id       |  status   | fraction_completed | rows | index_entries | system_records | bytes |
+--------------------+-----------+--------------------+------+---------------+----------------+-------+
| 352938237301293057 | succeeded |                  1 |    0 |             0 |              0 |     0 |
+--------------------+-----------+--------------------+------+---------------+----------------+-------+
(1 row)
~~~

Repeat the above steps for each table you want to import.

## Using `MYSQLOUTFILE`

{{site.data.alerts.callout_info}}
We support MySQL's "outfile" format, but we *strongly recommend* [using the faster and more robust `MYSQLDUMP` import](#using-mysqldump) if at all possible.
{{site.data.alerts.end}}

### Step 1. Dump the MySQL database to delimited text files

To export the entire `employees` database into the current directory, run the following command:

{% include copy-clipboard.html %}
~~~ shell
$ mysqldump -uroot -pfoo --tab=`pwd` employees --fields-enclosed-by='"' --fields-escaped-by='\'
~~~

For more information, see the [`mysqldump` documentation](https://dev.mysql.com/doc/refman/8.0/en/mysqldump-delimited-text.html).

## Step 2. Host the files where the cluster can access them

Each node in the CockroachDB cluster needs to have access to the files being imported.  There are several ways for the cluster to access the data; for a complete list of the types of storage [`IMPORT`][import] can pull from, see [Import File URLs][import#import-file-urls].

{{site.data.alerts.callout_success}}
We strongly recommend using cloud storage such as Amazon S3 or Google Cloud to host the files with data you want to import.
{{site.data.alerts.end}}

The files needed for this example are `employees.sql` (schema definition) and `employees.txt` (data).

### Step 3. Import the delimited files

To import the MySQL dump file for a table, issue an `IMPORT` statement like the one shown below.

You will need to look at the SQL schema definition for the table (in `employees.sql`) and translate the MySQL [`CREATE TABLE`](create-table.html) statement there into something CockroachDB understands.

{% include copy-clipboard.html %}
~~~ sql
> IMPORT TABLE employees (
    emp_no INT PRIMARY KEY,
    birth_date DATE NOT NULL,
    first_name VARCHAR NOT NULL,
    last_name VARCHAR NOT NULL,
    gender VARCHAR NOT NULL,
    hire_date DATE NOT NULL
  ) MYSQLOUTFILE DATA ('s3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=123&AWS_SECRET_ACCESS_KEY=456') WITH fields_enclosed_by = '"', fields_escaped_by = '\';
~~~

Success will look like:

~~~
+--------------------+-----------+--------------------+--------+---------------+----------------+----------+
|       job_id       |  status   | fraction_completed |  rows  | index_entries | system_records |  bytes   |
+--------------------+-----------+--------------------+--------+---------------+----------------+----------+
| 376504865832468481 | succeeded |                  1 | 300024 |             0 |              0 | 11534293 |
+--------------------+-----------+--------------------+--------+---------------+----------------+----------+
(1 row)
~~~

Repeat the above steps for each table you want to import.

## See also

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
[oracle]: migrate-from-oracle.html
[csv]: migrate-from-csv.html
