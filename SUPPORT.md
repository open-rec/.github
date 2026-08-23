# OpenRec Support

Use the repository that owns the failing behavior:

- `example`: installation, distribution compatibility, standalone/cluster startup, and failures
  involving multiple components;
- `rec-server`: recommendation/push APIs and online DAG behavior;
- `rec-console`: UI, operations, releases, analytics, and Airflow integration;
- `rec-algorithm`: offline jobs and algorithm output;
- `rank-engine`: model training/loading/inference;
- `data-processor`: Kafka, Redis, HBase, and Hive projections;
- `bigdata-platform`: infrastructure containers and monitoring;
- `sdk`: client behavior.

Open a GitHub issue for reproducible defects and feature proposals. Use GitHub Discussions, when
enabled, for design questions and general usage help. Security vulnerabilities must be reported
privately according to `SECURITY.md`.

For troubleshooting, include sanitized versions, mode, commands, health output, and relevant logs.
Maintainers cannot diagnose screenshots of partial errors, unspecified component versions, or logs
containing secrets and private customer data.
