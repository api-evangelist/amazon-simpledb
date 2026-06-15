---
title: "Automate Oracle PL/SQL to PostgreSQL migration with Amazon Bedrock and Strands Agents"
url: "https://aws.amazon.com/blogs/database/automate-oracle-pl-sql-to-postgresql-migration-with-amazon-bedrock-and-strands-agents/"
date: "2026-06-08"
author: "Kavita Vellala"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
In this post, you learn how to build a generative AI–powered migration assistant that helps automate portions of the last mile of code conversion. Using Anthropic’s Claude Sonnet 4.6 on Amazon Bedrock, the Strands Agents framework, and the AWS Knowledge MCP Server, you can automate the conversion and validation of PL/SQL objects against Amazon Aurora PostgreSQL-Compatible Edition. The assistant reads the AWS DMS SC assessment CSV, fetches live PL/SQL source from Oracle, converts each object, deploys the result to Aurora PostgreSQL through AWS Lambda, and runs automated tests, in a single pipel
