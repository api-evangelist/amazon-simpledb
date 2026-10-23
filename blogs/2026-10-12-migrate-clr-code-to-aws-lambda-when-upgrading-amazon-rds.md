---
title: "Migrate CLR code to AWS Lambda when upgrading Amazon RDS for SQL Server to 2025"
url: "https://aws.amazon.com/blogs/database/migrate-clr-code-to-aws-lambda-when-upgrading-amazon-rds-for-sql-server-to-2025/"
date: "2026-10-12"
author: "Raunak Gupta"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
After upgrading Amazon RDS for SQL Server to 2017 or later, CLR strict security leaves user-defined CLR assemblies unusable. This post shows how to migrate CLR logic with no native T-SQL equivalent to an AWS Lambda function, invoked from SQL Server 2025 through sp_invoke_external_rest_endpoint behind Amazon API Gateway.
