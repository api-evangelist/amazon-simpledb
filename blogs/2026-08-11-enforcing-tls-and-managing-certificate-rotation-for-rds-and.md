---
title: "Enforcing TLS and managing certificate rotation for RDS and Amazon Aurora PostgreSQL"
url: "https://aws.amazon.com/blogs/database/enforcing-tls-and-managing-certificate-rotation-for-rds-and-amazon-aurora-postgresql/"
date: "2026-08-11"
author: "Stefano D'Alessio"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
When an Amazon RDS or Amazon Aurora PostgreSQL certificate expires and client trust stores aren't updated, connections fail without warning. This post shows how to enforce TLS for all PostgreSQL connections, configure client-side certificate verification, and deploy automated monitoring that alerts you before certificate rotation events.
