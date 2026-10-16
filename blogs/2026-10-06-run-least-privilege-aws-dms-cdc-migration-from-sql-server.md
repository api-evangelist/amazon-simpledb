---
title: "Run least-privilege AWS DMS CDC migration from SQL Server"
url: "https://aws.amazon.com/blogs/database/run-least-privilege-aws-dms-cdc-migration-from-sql-server/"
date: "2026-10-06"
author: "Aritra Biswas"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Configure AWS DMS change data capture (CDC) from a self-managed SQL Server source without granting sysadmin to the endpoint login. This least-privilege pattern uses certificate-signed wrapper procedures, granular permissions, and AWS Secrets Manager to meet separation-of-duties requirements while preserving full CDC functionality.
