---
title: "Detect CDC failures faster with AWS DMS"
url: "https://aws.amazon.com/blogs/database/detect-cdc-failures-faster-with-aws-dms/"
date: "2026-08-05"
author: "Aritra Biswas"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
AWS DMS uses exponential backoff for recoverable errors, and default settings can let a change data capture (CDC) task retry silently for up to 30 minutes before failing. This post shows how to tune four recoverable-error settings so CDC tasks fail within minutes, and how to pair them with Amazon EventBridge and Amazon CloudWatch alerts.
