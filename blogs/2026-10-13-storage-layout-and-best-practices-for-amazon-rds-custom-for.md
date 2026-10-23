---
title: "Storage layout and best practices for Amazon RDS Custom for SQL Server"
url: "https://aws.amazon.com/blogs/database/storage-layout-and-best-practices-for-amazon-rds-custom-for-sql-server/"
date: "2026-10-13"
author: "Olushola Oladipupo"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Amazon RDS Custom for SQL Server gives you OS-level access, which also exposes internal storage volumes where the W: drive looks like a raw, uninitialized disk. This post explains the C:, D:, and W: drive architecture, how the drives map to Amazon EBS, and the safe disk operations and verification checks that keep you from accidentally destroying your data.
