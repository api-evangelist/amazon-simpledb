---
title: "Fix circular role dependencies before upgrading Amazon RDS and Amazon Aurora PostgreSQL"
url: "https://aws.amazon.com/blogs/database/resolve-circular-role-dependencies-during-upgrades-of-amazon-rds-for-postgresql-and-amazon-aurora/"
date: "2026-08-31"
author: "Ravi Teja Adabala"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Circular role dependencies can stall or roll back a major version upgrade of Amazon RDS for PostgreSQL or Amazon Aurora PostgreSQL when you move from PostgreSQL 14 or earlier to 15 or later. Learn why this happens, how to detect it with a single pre-upgrade query, and how to clear it before you upgrade.
