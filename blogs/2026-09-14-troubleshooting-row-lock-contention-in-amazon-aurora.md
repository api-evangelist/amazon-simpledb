---
title: "Troubleshooting row lock contention in Amazon Aurora PostgreSQL: Part 1 – Understanding row lock contention in PostgreSQL"
url: "https://aws.amazon.com/blogs/database/troubleshooting-row-lock-contention-in-amazon-aurora-postgresql-part-1-understanding-row-lock-contention-in-postgresql/"
date: "2026-09-14"
author: "Sameer Kumar"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Row lock contention can collapse database throughput during a flash sale even when CPU and I/O look healthy. In Part 1 of this series, learn how PostgreSQL row locking works and how to monitor lock contention in Amazon Aurora PostgreSQL and Amazon RDS for PostgreSQL using system views, the pgrowlocks extension, and the log_lock_waits parameter.
