---
title: "Amazon Aurora DSQL connections: Drivers, strings, and best practices"
url: "https://aws.amazon.com/blogs/database/amazon-aurora-dsql-connections-drivers-strings-and-best-practices/"
date: "Mon, 11 May 2026 20:48:43 +0000"
author: "Rob Petersen"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Connecting to Amazon Aurora DSQL requires a different approach than traditional PostgreSQL databases. Instead of long-lived passwords, you use short-lived IAM authentication tokens. Instead of static endpoints, you work with distributed cluster endpoints that route connections across Availability Zones.
