---
title: "Connect to Amazon RDS for Db2 from your laptop"
url: "https://aws.amazon.com/blogs/database/connect-to-amazon-rds-for-db2-from-your-laptop/"
date: "2026-05-04"
author: "Vikram Khatri"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
When you work with Amazon Relational Database Service (Amazon RDS for Db2) instances deployed in private subnets, connecting directly from your laptop requires a secure, manageable approach that doesn’t expose your database to the Internet. AWS Systems Manager Session Manager (SSM) solves this by acting as an encrypted tunnel—no public IP addresses, no SSH key management, and a full audit trail through AWS CloudTrail.
