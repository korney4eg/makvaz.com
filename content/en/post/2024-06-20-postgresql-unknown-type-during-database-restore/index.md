---
layout: post
title: Getting error with unknown variable type that used extension.
draft: false
---
_I faced an interesting issue when I tried to restore a database from a backup. The error said: `type "ltree" does not exist`. **ltree** is a [PostgreSQL extension](https://www.postgresql.org/docs/current/ltree.html), and this error can appear with any PostgreSQL extension. So how do we fix this error?_

<!--more-->
## TL;DR
When you create a backup with `pg_dump`, it generates a header with security options. One of those options is 
```sql
SELECT pg_catalog.set_config('search_path', '', false);
```
that cleans up `search_path` to prevent accessing objects, tables, types from other schemas, this is done to mitigate [CVE-2018-1058](https://wiki.postgresql.org/wiki/A_Guide_to_CVE-2018-1058:_Protect_Your_Search_Path). As a workaround you can export table that is using an extension to plain text backup and remove the line with setting `search_path`, then just import data from a patched file. The proper way would be to update all your tables and functions to use explicit paths to types, tables, and other objects, e.g. `public.ltree` instead of `ltree`.

## Longer version
To illustrate this issue we gonna use [materials from tedeh.net](https://tedeh.net/acyclic-and-directed-graph-in-postgres-with-the-ltree-extension/), which I recommend reading if you want to see `ltree` extension in practice.

### Setup

#### Install PostgreSQL client
Download and install [PostgreSQL client.](https://www.postgresql.org/download/)

#### Clone the repository
```bash
git clone https://github.com/korney4eg/postgresql-restore-example.git
cd postgresql-restore-example
```

#### Starting empty PostgreSQL server
```bash
docker-compose up -d
```

#### Create an example database
```bash
psql -h localhost -p 5432 -U root -c 'CREATE DATABASE example;'
```
As you can find in `docker-compose.yaml` root password for PostgreSQL is an "example".

#### Restoring example database
```bash
psql -h localhost -p 5432 -U root -d example < backup.sql
```

As you can see we got the error:
```
Password for user root: 
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
CREATE EXTENSION
COMMENT
CREATE FUNCTION
ALTER FUNCTION
CREATE FUNCTION
ALTER FUNCTION
CREATE FUNCTION
ALTER FUNCTION
SET
SET
CREATE TABLE
ALTER TABLE
CREATE SEQUENCE
ALTER SEQUENCE
ALTER SEQUENCE
ALTER TABLE
ALTER TABLE
CREATE INDEX
CREATE INDEX
CREATE TRIGGER
CREATE TRIGGER
ERROR:  operator does not exist: public.ltree = public.ltree
LINE 1: ...lic.employee FOR EACH ROW WHEN ((new.manager_path IS DISTINC...
                                                             ^
HINT:  No operator matches the given name and argument types. You might need to add explicit type casts.
ALTER TABLE
```
The error shows that PostgreSQL couldn't find any data regarding `ltree` operator.

#### Workaround

Let's recreate the whole server.
```bash
docker-compose down -v
rm -rf data/*
docker-compose up -d
```

Recreate database
```bash
psql -h localhost -p 5432 -U root -c 'CREATE DATABASE example;'
```

Now try to remove the line from `backup.sql`:
```sql
SELECT pg_catalog.set_config('search_path', '', false);
```

Try to restore one more time:
```bash
psql -h localhost -p 5432 -U root -d example < backup.sql
```

You see no error:
```
SET
SET
SET
SET
SET
SET
SET
SET
SET
CREATE EXTENSION
COMMENT
CREATE FUNCTION
ALTER FUNCTION
CREATE FUNCTION
ALTER FUNCTION
SET
SET
CREATE TABLE
ALTER TABLE
CREATE SEQUENCE
ALTER SEQUENCE
ALTER SEQUENCE
ALTER TABLE
COPY 5
 setval 
--------
      1
(1 row)

ALTER TABLE
CREATE INDEX
CREATE INDEX
CREATE TRIGGER
CREATE TRIGGER
ALTER TABLE
```

## Conclusion
This is not a secure way to mitigate the issue, but it works. I showed an example of a 5KB file example, but on production, it could be much more complicated. So do it at your risk and share your results in comments.
