---
title: "Migrate RDS and Aurora logs to CloudWatch Infrequent Access"
url: "https://aws.amazon.com/blogs/database/migrate-rds-and-aurora-logs-to-cloudwatch-infrequent-access/"
date: "2026-08-11"
author: "Ryan Moore"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Organizations running Amazon RDS and Amazon Aurora often pay full CloudWatch Logs ingestion rates for database logs they rarely access. This post shows how to build an automated, tag-driven solution that migrates RDS and Aurora CloudWatch log groups from the Standard to the Infrequent Access log class and cuts log ingestion costs by about 50%.
