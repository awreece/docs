---
title: Migrate from Postgres
summary: Learn how to migrate data from Postgres into a CockroachDB cluster.
toc: true
---

This page has instructions for migrating from Postgres to CockroachDB using [`IMPORT`](import.html).

The examples below use the [employees data set](https://github.com/datacharmer/test_db) that is also used in the [MySQL docs](https://dev.mysql.com/doc/employee/en/).  The data set was imported into Postgres from MySQL using [pgloader](https://pgloader.io).

## Using Postgres `COPY`

### Step 1. Dump the Postgres database

To dump the `employees` table from a Postgres database also named `employees`, run the [`pg_dump`](https://www.postgresql.org/docs/current/static/app-pgdump.html) command below.

{% include copy-clipboard.html %}
~~~ shell
$ pg_dump -t employees employees > /tmp/employees.sql
~~~

### Step 2. Edit the Postgres dump file

Open the Postgres dump file and copy the `CREATE TABLE` statement somewhere safe (such as a temp file).  We will use this later when we create the [`IMPORT TABLE`][import] statement that pulls the data into CRDB.

Then, remove everything from the beginning of the file to the first line after `COPY` which should contain only data, e.g., lines that look like

```
--
-- PostgreSQL database dump
--
...
COPY employees.employees (emp_no, birth_date, first_name, last_name, gender, hire_date) FROM stdin;
-- DELETE EVERYTHING ABOVE THIS LINE
10001	1953-09-02	Georgi	Facello	M	1986-06-26
...
499999	1958-05-01	Sachin	Tsukuda	M	1997-11-30
-- DELETE EVERYTHING BELOW THIS LINE
\.
...
--
-- PostgreSQL database dump complete
--
```

Then, navigate to the end of the file, and remove everything between the last line of data and EOF.

### Step 3. `IMPORT` the Postgres dump file

Next, use the Postgres `CREATE TABLE` statement we saved in the last step. You will need to translate it into an [`IMPORT TABLE`](import.html) statement CockroachDB understands.

To import the Postgres dump file, issue an `IMPORT` statement like the one below.

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
  PGCOPY DATA ('nodelocal:///employees.sql');
~~~

## Using Postgres `INSERT`

### Step 1. Dump the Postgres database

To dump the `employees` table from a Postgres database also named `employees`, run the [`pg_dump`](https://www.postgresql.org/docs/current/static/app-pgdump.html) command below.

{% include copy-clipboard.html %}
~~~ shell
$ pg_dump --inserts --no-privileges --no-owner -t employees  employees > /tmp/employees.sql
~~~

The following options are required:

 Option            | Description
-------------------+--------------------------------------------------------------------------------------------------------------------------------------
 `--inserts`       | Required until CockroachDB implements support for the Postgres [`COPY`](https://www.postgresql.org/docs/current/static/sql-copy.html) statement.
 `--no-privileges` | Required until CockroachDB implements support for Postgres's access privilege statements.

## Step 2. Host the files where the cluster can access them

Each node in the CockroachDB cluster needs to have access to the files being imported.  There are several ways for the cluster to access the data; for a complete list of the types of storage [`IMPORT`][import] can pull from, see [Import File URLs][import#import-file-urls].

{{site.data.alerts.callout_success}}
We strongly recommend using cloud storage such as Amazon S3 or Google Cloud to host the files with data you want to import.
{{site.data.alerts.end}}

The file needed for this example is `employees.sql`.

### Step 2. `IMPORT` the Postgres dump file

You will need to look at the dump file and translate the Postgres [`CREATE TABLE`](create-table.html) statement there into an [`IMPORT TABLE`](import.html) statement CockroachDB understands.

For this data set, the Postgres dump file required one edit: the type of the `gender` column had to be changed from `employees.employees_gender` to [`STRING`](string.html) since Postgres represented the employee's gender using a [`CREATE TYPE`](https://www.postgresql.org/docs/10/static/sql-createtype.html) statement that is not yet supported by CockroachDB. If you'd rather not edit the Postgres dump file by hand, you can run the following command (which was only tested for this example):

{% include copy-clipboard.html %}
~~~ shell
$ perl -i.bak -lapE 's/gender employees\.employees_gender/gender STRING/' employees.sql
~~~

To import the Postgres dump file, issue an `IMPORT` statement like the one below.

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
  PGDUMP DATA ('s3://your-external-storage/employees.sql?AWS_ACCESS_KEY_ID=ACCESSKEY&AWS_SECRET_ACCESS_KEY=SECRET');
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

Finally, confirm that despite needing to edit the `gender` column's data type in the Postgres dump file, the column was populated correctly during the import:

{% include copy-clipboard.html %}
~~~ sql
SELECT * FROM employees LIMIT 10;
~~~

~~~
+--------+---------------------------+------------+-----------+--------+---------------------------+
| emp_no |        birth_date         | first_name | last_name | gender |         hire_date         |
+--------+---------------------------+------------+-----------+--------+---------------------------+
|  10001 | 1953-09-02 00:00:00+00:00 | Georgi     | Facello   | M      | 1986-06-26 00:00:00+00:00 |
|  10002 | 1964-06-02 00:00:00+00:00 | Bezalel    | Simmel    | F      | 1985-11-21 00:00:00+00:00 |
|  10003 | 1959-12-03 00:00:00+00:00 | Parto      | Bamford   | M      | 1986-08-28 00:00:00+00:00 |
|  10004 | 1954-05-01 00:00:00+00:00 | Chirstian  | Koblick   | M      | 1986-12-01 00:00:00+00:00 |
|  10005 | 1955-01-21 00:00:00+00:00 | Kyoichi    | Maliniak  | M      | 1989-09-12 00:00:00+00:00 |
|  10006 | 1953-04-20 00:00:00+00:00 | Anneke     | Preusig   | F      | 1989-06-02 00:00:00+00:00 |
|  10007 | 1957-05-23 00:00:00+00:00 | Tzvetan    | Zielinski | F      | 1989-02-10 00:00:00+00:00 |
|  10008 | 1958-02-19 00:00:00+00:00 | Saniya     | Kalloufi  | M      | 1994-09-15 00:00:00+00:00 |
|  10009 | 1952-04-19 00:00:00+00:00 | Sumant     | Peac      | F      | 1985-02-18 00:00:00+00:00 |
|  10010 | 1963-06-01 00:00:00+00:00 | Duangkaew  | Piveteau  | F      | 1989-08-24 00:00:00+00:00 |
+--------+---------------------------+------------+-----------+--------+---------------------------+
(10 rows)
~~~

To load the data from local files, you have 2 options:

- Start up a [local HTTP server](create-a-file-server.html#using-caddy-as-a-file-server) to serve the files, and point `IMPORT ... PGDUMP` at the server's URL. This is the recommended option.

- Create an `extern` subdirectory on each node and copy the Postgres dump files there. Note that you will need to copy the dump files onto every node, since CockroachDB may execute the `IMPORT` statement on any of the nodes.

If you decide to load the data from the `extern` subdirectory, you will need to use [`IMPORT`'s `nodelocal` URL scheme](import.html#import-file-urls) as shown below.

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
  PGDUMP DATA ('nodelocal:///employees_table.sql');
~~~

## Configuration Options

The new `max_row_size` option overrides default limits on line size for `IMPORT ... PGDUMP` and `PGCOPY`, e.g.,

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
  PGCOPY DATA ('nodelocal:///employees_table.sql') WITH max_row_size = '5B';
~~~

## Other Considerations

## See also

- [`IMPORT`](import.html)
- [SQL Dump (Export)](sql-dump.html)
- [Back up Data](back-up-data.html)
- [Restore Data](restore-data.html)
- [Use the Built-in SQL Client](use-the-built-in-sql-client.html)
- [Other Cockroach Commands](cockroach-commands.html)

<!-- Links -->

[import]: import.html
