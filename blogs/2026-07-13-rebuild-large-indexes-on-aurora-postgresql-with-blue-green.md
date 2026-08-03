---
title: "Rebuild large indexes on Aurora PostgreSQL with Blue/Green Deployments"
url: "https://aws.amazon.com/blogs/database/rebuild-large-indexes-on-aurora-postgresql-with-blue-green-deployments/"
date: "2026-07-13"
author: "Santosh Mishra"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
In this post, we show how to rebuild large indexes on Amazon Aurora PostgreSQL by combining Amazon Aurora Blue/Green Deployments with Aurora Optimized Reads. By performing the reindex on the green (staging) environment with a Non-Volatile Memory express (NVMe)-backed instance class, the sort phase uses fast local storage instead of Amazon EBS over the network, and you avoid impacting production workloads.
