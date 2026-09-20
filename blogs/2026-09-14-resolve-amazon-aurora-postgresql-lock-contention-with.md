---
title: "Resolve Amazon Aurora PostgreSQL lock contention with Database Insights: Part 2"
url: "https://aws.amazon.com/blogs/database/resolve-amazon-aurora-postgresql-lock-contention-with-database-insights-part-2/"
date: "2026-09-14"
author: "Sameer Kumar"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Part 1 showed how row lock contention degrades Amazon Aurora PostgreSQL throughput. In Part 2, use Amazon CloudWatch Database Insights and its Lock Tree to pinpoint blocking sessions, then resolve contention with query termination, timeout parameters, and architectural patterns such as SKIP LOCKED and row splitting that restore throughput.
