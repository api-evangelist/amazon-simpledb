---
title: "Implement a correctness-safe Bloom filter lookup with Amazon ElastiCache for Valkey and Amazon Aurora PostgreSQL"
url: "https://aws.amazon.com/blogs/database/implement-a-correctness-safe-bloom-filter-lookup-with-amazon-elasticache-for-valkey-and-amazon-aurora-postgresql/"
date: "2026-09-17"
author: "Chintan Agrawal"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
This post shows how to compose a Bloom filter with an exact-match cache and a relational source of truth into a three-tier, correctness-safe membership lookup using Amazon ElastiCache for Valkey and Amazon Aurora PostgreSQL, serving sub-millisecond decisions at peak throughput without false-positive risk.
