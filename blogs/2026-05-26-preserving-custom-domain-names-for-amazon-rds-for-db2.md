---
title: "Preserving custom domain names for Amazon RDS for Db2"
url: "https://aws.amazon.com/blogs/database/preserving-custom-domain-names-for-amazon-rds-for-db2/"
date: "2026-05-26"
author: "Vikram Khatri"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
In this post, we introduce a modular Terraform template, published in the aws-samples/sample-rds-db2-tools repository, that lets your applications keep their existing custom domain names and ports while preserving end-to-end TLS encryption to Amazon RDS for Db2. The template deploys a Server Name Indication (SNI) based TLS proxy that forwards encrypted traffic without ever decrypting it.
