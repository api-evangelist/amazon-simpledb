---
title: "How HotelTrader cut inter-AZ cost 95% and latency by 49% with Valkey GLIDE on Amazon ElastiCache"
url: "https://aws.amazon.com/blogs/database/how-hoteltrader-cut-inter-az-cost-95-and-latency-by-49-with-valkey-glide-on-amazon-elasticache/"
date: "Wed, 13 May 2026 19:50:39 +0000"
author: "Sundeep Kumar S"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
In this post, you learn how HotelTrader reduced inter-availability zone data transfer costs by 95% and improved average latency by 49% by migrating from the Redis Lettuce client to Valkey GLIDE on Amazon ElastiCache. The post walks through how HotelTrader identified hidden cross-AZ data transfer costs in their multi-AZ ElastiCache cluster, implemented Valkey GLIDE's AZ-affinity read strategy to route requests to local replicas, optimized throughput with request batching, and executed a zero-downtime migration using A/B testing over 15 days.
