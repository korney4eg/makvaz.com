---
layout: post
title: What I have learned after working with PosgreSQL databases for half a year
draft: false
archives: "2025"
tags: [postgresql, howto]
---
_Last half a year I've been working with PosgreSQL database and learn many things. In this post you can find tips and my findings during this time._

<!--more-->

## Better backup and restore

Let's restore database from previous [post](/2024/06/20/postgresql-unknown-type-during-database-restore-p1/) and use it as initial point. So now let's make proper backup and split it into separate parts:

* part with pre-schema
* part with post-schema
* part with data

```bash
pg_dump postgresql://$SLAVE_USERNAME@${SOURCE}/api --no-owner --no-privileges --schema-only --section=pre-data --file=${PRE_SCHEMA} --verbose
pg_dump postgresql://$SLAVE_USERNAME@${SOURCE}/api --no-owner --no-privileges --schema-only --section=post-data --format=d --file=${POST_SCHEMA} --verbose
```

## List running queries

```sql
SELECT
    pid, age(clock_timestamp(), query_start), usename, query
FROM
    pg_stat_activity
WHERE
    query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY
    query_start desc;
```

## Conclusion
This is not a secure way to mitigate the issue, but it works. I showed an example of a 5KB file example, but on production, it could be much more complicated. So do it at your risk and share your results in comments.
