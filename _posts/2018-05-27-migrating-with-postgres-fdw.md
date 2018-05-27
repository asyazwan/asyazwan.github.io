---
title: "Migrating with Postgres FDW"
categories: article
tags: dev postgresql postgres-fdw
---

In a situation where you are migrating data from one provider to another and do not want to use intermediate node, foreign data wrapper helps. Foreign data wrapper / FDW allows us to connect to remote data sources from another data source giving the illusion it's local to us.

{% include figure image_path="/assets/images/2018/fdw-migration.svg" alt="migration illustration" caption="Migration with & without intermediary" %}

Earlier versions of Postgres only allows read-only data. As of Postgres 9.3, writing is available so if you are doing write operations take note of that.
{: .notice--warning}

My use case was this: I wanted to transfer gigabytes of data from provider A in Singapore region to another provider B in EU region. These providers provide DB access only, no OS-level access. So it wasn't possible to dump as usual and there's the issue of size too. To make my workstation the intermediate node (import from A - export to B) would also be a huge waste of bandwidth and space (about 400 GB total). I also don't have the best internet connection. Enter FDW.

First create the extension:

```sql
CREATE EXTENSION postgres_fdw;
```

Then create the connection:

```sql
CREATE SERVER myremotedb
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'myremotedb.providera.com', port '5432', dbname 'mydb', sslmode 'require', fetch_size '250000');
```

Important gotcha: find the right `fetch_size` for you. The default is 100 rows and extremely small for tables with millions of rows thus can cause big overhead depending on latency between the two servers.
{: .notice--warning}

You may exclude or include any server options as you require.

Now that we have the server, next we declare the user mapping:

```sql
CREATE USER MAPPING FOR newuser1
SERVER myremotedb
OPTIONS (user 'olduser1', password 'password1');
```

Lastly, the schema:

```sql
CREATE SCHEMA oldschema;

IMPORT FOREIGN SCHEMA public FROM SERVER myremotedb INTO oldschema;
```

Now you can query the remote database from the new one as if it's a local db:

```sql
SELECT * FROM oldschema.table1;

-- Migrate: (obviously their structure must be the same)
INSERT INTO public.table1 SELECT * FROM oldschema.table1;
```

## Results

FDW is usually not the most performant way to do things so it's rarely the first thing I look at. But here it proved to be useful since my migration wasn't so time-critical. With this setup I was able to move over 500k rows, 250 MB data in 30s (as opposed to 20m with default `fetch_size`!). Moving 2.7m rows, 8 GB data took 80 minutes, which is not so bad.

However, if intermediate node is a non-issue for you, it's better to use it. In my initial tests using `\copy` to dump CSV did over 1k rows/s on average, which is much faster than FDW. But of course that's exluding importing the dump to the new db.

Recommended reads:
- [List of FDW](https://wiki.postgresql.org/wiki/Foreign_data_wrappers) (More on [PGXN](https://pgxn.org/tag/fdw/))
- [Intro to Postgres FDW](http://www.craigkerstiens.com/2016/09/11/a-tour-of-fdws/)
