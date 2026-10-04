---
title: "Migrate SQL Server multi-result-set procedures to PostgreSQL"
url: "https://aws.amazon.com/blogs/database/migrate-sql-server-multi-result-set-procedures-to-postgresql/"
date: "2026-09-21"
author: "Ken Zhang"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
SQL Server stored procedures can return multiple result sets from one call, but PostgreSQL cannot. This post presents two PostgreSQL-native alternatives to refcursors, session-scoped temporary tables and JSON aggregation, compares both against a refcursor baseline, and shows how to implement and validate each in .NET and Npgsql.
