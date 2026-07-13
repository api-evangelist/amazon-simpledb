---
title: "Logical replication improvements in Amazon RDS for PostgreSQL 18"
url: "https://aws.amazon.com/blogs/database/logical-replication-improvements-in-amazon-rds-for-postgresql-18/"
date: "2026-07-07"
author: "Ramdas Gutlapalli"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
In this post, we demonstrate how to use the PostgreSQL 18 logical replication improvements on RDS for PostgreSQL: replicating STORED generated columns with the publish_generated_columns parameter, monitoring conflicts through the new counters in pg_stat_subscription_stats, verifying that parallel streaming is enabled by default, toggling two-phase commit on a running subscription, and configuring idle_replication_slot_timeout for automatic slot cleanup. These features are available on RDS for PostgreSQL 18.0 and later and Aurora PostgreSQL.
