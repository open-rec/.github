# OpenRec

OpenRec is an open-source recommendation system built around a configurable online serving DAG,
streaming feature pipelines, distributed offline algorithms, and an operational control plane.
It can run as a minimal standalone stack for development or as a complete cluster workflow for
production-oriented integration and scheduling.

## Projects

| Project | Responsibility |
|---|---|
| [rec-server](https://github.com/open-rec/rec-server) | Java online recommendation service and configurable serving DAG |
| [rec-algorithm](https://github.com/open-rec/rec-algorithm) | Spark offline recall, feature, and ranking jobs |
| [rank-engine](https://github.com/open-rec/rank-engine) | Python model inference service |
| [rec-console](https://github.com/open-rec/rec-console) | Recall-index control plane and operations UI |
| [data-processor](https://github.com/open-rec/data-processor) | Kafka streaming pipelines implemented with Spark and Flink |
| [bigdata-platform](https://github.com/open-rec/bigdata-platform) | Reusable storage, messaging, compute, and scheduling infrastructure |
| [example](https://github.com/open-rec/example) | Standalone and cluster composition roots with end-to-end verification |
| [sdk](https://github.com/open-rec/sdk) | Client SDKs for OpenRec services |

## Standalone architecture

Standalone mode is the smallest complete recommendation chain. It is intended for local
development, algorithm verification, and integration testing on one machine.

```mermaid
flowchart LR
    User[User or application] --> Web[Web Demo or SDK]
    Web --> API[rec-server<br/>standalone profile]

    subgraph Storage[bigdata-platform standalone]
        Redis[(Redis<br/>entities · events · exposure filters)]
        ES[(Elasticsearch<br/>recall and vector indexes)]
    end

    API <-->|entities · behavior · filters| Redis
    API <-->|recall · vector search| ES
    API -->|recommendations| Web

    Loader[Example data loader] --> Redis
    Loader --> ES
```

Key characteristics:

- `rec-server` writes pushed data directly to Redis.
- Hot, new, i2i, and embedding recall data are queried from Elasticsearch.
- Redis retains online entities, behavior, blacklist, and exposure-filter state.
- Ranking runs in bypass mode; `rank-engine` is not required.
- Kafka, distributed processing, Airflow, and `rec-console` are not required.

The internal recall, filtering, combination, and operation flow is defined by the configurable
serving DAG in [rec-server](https://github.com/open-rec/rec-server) and is intentionally omitted
from this deployment-level diagram.

Start the complete standalone example:

```shell
./example/example_standalone/start.sh
```

See the [standalone guide](https://github.com/open-rec/example/tree/master/example_standalone) for
prerequisites, endpoints, smoke verification, and troubleshooting.

## Cluster architecture

Cluster mode is the composition root for the complete recommendation lifecycle. It combines
online serving, real-time ingestion, distributed offline computation, scheduled publishing, model
inference, and recall-index operations.

```mermaid
flowchart TB
    Client[User · Web Demo · SDK] --> RecServer[rec-server<br/>cluster profile]

    subgraph Online[Online recommendation path]
        direction LR
        RecallDAG[Serving DAG<br/>recall · filter · combine]
        RecallDAG --> Rank[rank-engine<br/>model inference]
        Rank --> Result[Ranked recommendations]
        Result --> Response[Response to caller]
    end

    subgraph Streaming[Real-time ingestion path]
        direction LR
        Kafka[(Kafka)]
        Kafka --> Processor[Spark data-processor]
    end

    subgraph Storage[Shared serving and analytical storage]
        direction LR
        Redis[(Redis<br/>features · behavior · filters)]
        Hive[(Hive entity and event tables)]
        ES[(Elasticsearch<br/>versioned recall indexes)]
    end

    subgraph Offline[Daily offline recall path]
        direction LR
        Airflow[Airflow<br/>bootstrap and daily DAGs] --> Runner[rec-algorithm runner]
        Runner -->|spark-submit| Spark[Spark cluster<br/>hot · new · i2i jobs]
        Spark -->|prepare and activate| Console[rec-console]
    end

    RecServer -->|recommend request| RecallDAG
    RecServer -->|push API| Kafka

    RecallDAG -->|online state| Redis
    RecallDAG -->|active recall aliases| ES
    Processor -->|online features| Redis
    Processor -->|entity and event data| Hive

    Hive <-->|read daily partitions · write result tables| Spark
    Spark -->|bulk-write staging index| ES
    Console -->|create · validate · switch · retain · rollback| ES
```

The recall release protocol keeps online serving independent from index deployment:

```mermaid
sequenceDiagram
    participant A as Airflow
    participant R as rec-algorithm runner
    participant S as Spark recall job
    participant C as rec-console
    participant E as Elasticsearch
    participant O as rec-server

    A->>R: Submit daily hot, new, or i2i job
    R->>S: spark-submit with business date and revision
    S->>C: Prepare versioned staging index
    C->>E: Create mapping and physical index
    S->>E: Bulk-write recall documents
    S->>C: Activate with expected document count
    C->>E: Refresh and validate count
    C->>E: Atomically switch active alias
    O->>E: Continue querying the stable active alias
    C->>E: Retain active and rollback versions
```

Key characteristics:

- Kafka and Spark decouple online push traffic from feature persistence.
- HDFS and Hive provide partitioned source and result tables for offline jobs.
- Airflow owns dependency orchestration and daily scheduling without managing Docker containers.
- `rec-console` owns index creation, validation, activation, retention, explicit switching, and
  emergency rollback; `rec-server` remains read-only and version-agnostic.
- `rank-engine` provides online model inference and can evolve independently from serving DAGs.
- The default recall retention policy keeps the active version plus one rollback version.

Start and verify the complete cluster:

```shell
./example/example_cluster/start.sh
```

Main endpoints after startup:

| Service | Endpoint |
|---|---|
| Web Demo | `http://<host>:12345` |
| rec-server | `http://<host>:13579` |
| rank-engine | `http://<host>:8123` |
| rec-console | `http://<host>:8095` |
| Airflow | `http://<host>:8091` |
| Spark | `http://<host>:8083` |
| Flink | `http://<host>:8087` |

See the [cluster guide](https://github.com/open-rec/example/tree/master/example_cluster) for the
full startup sequence, service ownership, DAG behavior, and troubleshooting.

## Deployment comparison

| Concern | Standalone | Cluster |
|---|---|---|
| Primary use | Local development and smoke testing | Complete distributed integration |
| Infrastructure | Redis and Elasticsearch | Kafka, HDFS, Hive, Spark, Flink, Redis, Elasticsearch, Airflow |
| Push path | `rec-server` writes Redis directly | `rec-server` publishes to Kafka |
| Streaming processor | Not required | Spark data-processor |
| Offline scheduling | Not required | Airflow and Spark recall jobs |
| Recall index control | Example bootstrap data | `rec-console` version lifecycle |
| Ranking | Bypassed | `rank-engine` inference |
| Online recall contract | Elasticsearch aliases and vector indexes | Same online contract |

Both modes use the same `rec-server` image, recommendation DAG, storage contracts, sample data,
and Web Demo. Runtime profiles change deployment behavior without changing the recommendation API.
