---
title: "Resolving PostgreSQL replication lag with heartbeat tables in change data capture scenarios"
url: "https://aws.amazon.com/blogs/database/resolving-postgresql-replication-lag-with-heartbeat-tables-in-change-data-capture-scenarios/"
date: "2026-09-30"
author: "Donghua Luo"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Replication lag during a change data capture (CDC) migration can be counterintuitive: the source is busy, yet replication falls behind and WAL piles up. This post shows how to resolve PostgreSQL replication lag with heartbeat tables using AWS DMS or Debezium, and how to monitor replication slot health with Amazon CloudWatch and SQL diagnostics.
