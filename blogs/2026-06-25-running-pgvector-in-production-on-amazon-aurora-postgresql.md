---
title: "Running pgvector in production on Amazon Aurora PostgreSQL"
url: "https://aws.amazon.com/blogs/database/running-pgvector-in-production-on-amazon-aurora-postgresql/"
date: "2026-06-25"
author: "Stefan Aichholzer"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Running pgvector on Amazon Aurora PostgreSQL gives you a production-grade vector store on a database you already know, backed by the operational tooling, high availability, and scaling behaviour of Amazon Aurora. Production traffic does introduce a predictable set of operational considerations: query latency as the corpus grows, recall on filtered vector searches, memory headroom during index builds, and connection behaviour under load. This post is scoped to the database operations that keep the RAG retrieval layer healthy.
