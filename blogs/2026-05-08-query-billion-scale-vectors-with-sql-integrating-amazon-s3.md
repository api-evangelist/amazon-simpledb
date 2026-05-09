---
title: "Query billion-scale vectors with SQL: Integrating Amazon S3 Vectors and Aurora PostgreSQL"
url: "https://aws.amazon.com/blogs/database/query-billion-scale-vectors-with-sql-integrating-amazon-s3-vectors-and-aurora-postgresql/"
date: "2026-05-08"
author: "Shayon Sanyal"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
If you already manage relational data in Amazon Aurora PostgreSQL and need to add similarity search over large embedding collections without migrating everything into your database, this post is for you. Aurora PostgreSQL with pgvector excels at low-latency similarity searches on database-resident vectors, Amazon S3 Vectors provides economical storage for massive vector datasets that may reach hundreds of millions or billions of embeddings. By connecting these services through AWS Lambda, you get a familiar SQL interface for both vector search and relational joins.
