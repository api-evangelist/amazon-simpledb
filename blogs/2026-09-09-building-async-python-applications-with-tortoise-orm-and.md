---
title: "Building async Python applications with Tortoise ORM and Amazon Aurora DSQL"
url: "https://aws.amazon.com/blogs/database/building-async-python-applications-with-tortoise-orm-and-amazon-aurora-dsql/"
date: "2026-09-09"
author: "Lasita Bhattacharya"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Build a high-concurrency async Python rideshare application with Tortoise ORM and Amazon Aurora DSQL. This post walks through the key adaptations: UUID primary keys, IAM-authenticated asyncpg connections with a connection-pool patch, individual DDL execution, and optimistic concurrency control (OCC) retry logic.
