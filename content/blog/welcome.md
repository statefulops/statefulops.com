---
title: "Welcome to the Statefulops blog"
date: 2026-10-22T09:00:00+01:00
description: "What this blog is about, and what you can expect to read here."
tags: ["meta"]
---

Welcome! This is the blog for Statefulops, an independent consultancy focused on databases and the stateful systems around them.

<!--more-->

## What I'll be writing about

Expect practical, experience-led posts on topics such as:

- Performance tuning and query optimisation
- High availability, backups and disaster recovery
- Running databases on Kubernetes
- Migrations, upgrades and the lessons learned along the way

## A taste of what's to come

Posts will often include code and configuration you can use directly. For example, a quick way to find the largest tables in a PostgreSQL database:

```sql
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

{{< callout type="tip" >}}
Subscribe to the [RSS feed](/blog/index.xml) to get new posts as they're published.
{{< /callout >}}

If there's a topic you'd like me to cover, [let me know](/contact/).
