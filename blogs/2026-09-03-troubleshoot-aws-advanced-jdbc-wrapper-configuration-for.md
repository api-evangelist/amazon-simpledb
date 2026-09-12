---
title: "Troubleshoot AWS Advanced JDBC Wrapper configuration for Aurora Global Database write forwarding"
url: "https://aws.amazon.com/blogs/database/troubleshoot-aws-advanced-jdbc-wrapper-configuration-for-aurora-global-database-write-forwarding/"
date: "2026-09-03"
author: "Hema Saminathan"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Configuring the AWS Advanced JDBC Wrapper for Amazon Aurora Global Database with write forwarding requires Region-specific settings, and misconfiguration causes latency spikes and connection failures. This post walks through the correct dialect, plugins, host patterns, and write forwarding settings for the primary and secondary Regions.
