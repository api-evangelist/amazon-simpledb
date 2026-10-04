---
title: "Megabytes in milliseconds: How FireTV uses parallel queries and vertical partitioning to serve millions of customers in Amazon DynamoDB"
url: "https://aws.amazon.com/blogs/database/megabytes-in-milliseconds-how-firetv-uses-parallel-queries-and-vertical-partitioning-to-serve-millions-of-customers-in-amazon-dynamodb/"
date: "2026-09-23"
author: "Mohit Agarwal"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
How Amazon FireTV redesigned its Continue Watching watch-progress data model on Amazon DynamoDB with vertical partitioning and hash-prefixed sort keys for parallel reads, removing item-size limits, cutting write costs by 97%, and keeping reads under 50 milliseconds at any profile size.
