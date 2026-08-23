# Contributing to OpenRec

Thank you for helping improve OpenRec. The project is split into focused repositories so changes can
be reviewed, tested, and released by the people who own the affected contract.

## Choose the right repository

- Installation, distribution manifests, complete examples, and cross-component compatibility:
  [`example`](https://github.com/open-rec/example)
- Online API, protocol, serving graph, recall/filter/rank nodes: `rec-server`
- Operations UI and control-plane APIs: `rec-console`
- Offline recall, feature, embedding, and ranking jobs: `rec-algorithm`
- Rank-model inference and loading: `rank-engine`
- Kafka projections and distributed processing: `data-processor`
- Docker infrastructure and observability: `bigdata-platform`
- Client integrations: `sdk`

For a defect spanning repositories, open one issue in `example`, list every affected component, and
link component pull requests back to that issue.

## Before opening an issue

Search existing issues and documentation first. Include:

- OpenRec distribution version or component commit SHA;
- standalone or cluster mode;
- operating system, architecture, Docker, Java, Python, and Maven versions;
- exact command and the smallest reproducible configuration;
- sanitized logs and expected versus actual behavior.

Never attach credentials, tokens, private data, or unredacted production logs. Security issues must
follow `SECURITY.md` rather than a public issue.

## Pull requests

1. Discuss large API, schema, storage, or architecture changes in an issue first.
2. Keep each pull request scoped to one repository and concern.
3. Follow the style and test conventions documented in that repository.
4. Add or update focused tests and documentation.
5. For contract changes, update the `example` distribution and its end-to-end acceptance checks.
6. Complete the pull request template, including commands run and compatibility impact.

Commit subjects should be short and imperative, for example `add recall release check` or
`fix graph timeout propagation`.

## Compatibility-sensitive changes

The following require explicit versioning, migration notes, and end-to-end validation:

- recommendation or push API payloads;
- Kafka mutation envelopes;
- Redis keys or Elasticsearch mappings and aliases;
- Hive/HBase schemas;
- serving-graph JSON;
- model artifact metadata;
- environment variables, ports, health checks, or Compose service names.

Avoid silently changing a shared contract. Prefer additive changes, schema versions, deprecation
periods, and a tested rollback path.

## Review and conduct

Maintainers may ask for a smaller change, additional evidence, or a compatibility test. Reviews
focus on behavior, operability, security, and long-term maintenance. Participation is governed by
the OpenRec Code of Conduct.
