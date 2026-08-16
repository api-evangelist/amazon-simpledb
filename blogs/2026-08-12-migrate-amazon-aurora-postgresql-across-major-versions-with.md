---
title: "Migrate Amazon Aurora PostgreSQL across major versions with active Debezium CDC connectors using native logical replication"
url: "https://aws.amazon.com/blogs/database/migrate-amazon-aurora-postgresql-across-major-versions-with-active-debezium-cdc-connectors-using-native-logical-replication/"
date: "2026-08-12"
author: "Arko Dutta"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
Standard upgrade paths break active Debezium CDC replication slots on Amazon Aurora PostgreSQL, forcing hours-long re-snapshots. This post shows how to use native PostgreSQL logical replication to bridge a source and target cluster and cut your Debezium connectors over to the new major version with a brief, measured write pause and no re-snapshot.
