---
title: "Migrate multilingual full-text search from SQL Server to PostgreSQL"
url: "https://aws.amazon.com/blogs/database/migrate-multilingual-full-text-search-from-sql-server-to-postgresql/"
date: "2026-08-20"
author: "Ken Zhang"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Migrating full-text search from SQL Server to PostgreSQL can silently change results because the engines handle text, linguistics, and accents differently. This post shows how to reproduce SQL Server full-text search on Amazon Aurora PostgreSQL and Amazon RDS for PostgreSQL, covering collation, tokenization, accent-insensitive search, and synonyms.
