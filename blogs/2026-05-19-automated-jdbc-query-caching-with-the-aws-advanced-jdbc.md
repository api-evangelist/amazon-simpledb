---
title: "Automated JDBC query caching with the AWS Advanced JDBC Wrapper"
url: "https://aws.amazon.com/blogs/database/automated-jdbc-query-caching-with-the-aws-advanced-jdbc-wrapper/"
date: "2026-05-19"
author: "Qu Chen"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Today, we’re announcing the Remote Query Cache Plugin for the AWS Advanced JDBC Wrapper. The plugin handles query caching automatically. It intercepts JDBC queries, caches results in Amazon ElastiCache for Valkey, and serves subsequent identical queries from cache.
