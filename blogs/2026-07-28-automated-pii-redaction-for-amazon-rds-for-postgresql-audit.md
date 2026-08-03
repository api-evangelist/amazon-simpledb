---
title: "Automated PII redaction for Amazon RDS for PostgreSQL audit logs"
url: "https://aws.amazon.com/blogs/database/automated-pii-redaction-for-amazon-rds-for-postgresql-audit-logs/"
date: "2026-07-28"
author: "Harish Bannai"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
In this post, we show you how to deploy a serverless pipeline that creates an irreversibly redacted, queryable archive of Amazon RDS for PostgreSQL audit logs. The pipeline permanently removes Social Security numbers (SSNs), credit cards, email, names, and more than 30 types of personally identifiable information (PII) before storing clean logs in Amazon S3 for query through Amazon Athena.
