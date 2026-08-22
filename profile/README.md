# OpenRec

OpenRec is an open-source recommendation system built around a configurable online serving DAG,
streaming feature pipelines, distributed offline algorithms, and an operational control plane.
It can run as a minimal standalone stack for development or as a complete cluster workflow for
production-oriented integration and scheduling.

## Projects

| Project | Responsibility |
|---|---|
| [rec-server](https://github.com/open-rec/rec-server) | Java online recommendation service and configurable serving DAG |
| [rec-algorithm](https://github.com/open-rec/rec-algorithm) | Local and Spark recall, feature, and ranking jobs |
| [rank-engine](https://github.com/open-rec/rank-engine) | Python model inference service |
| [rec-console](https://github.com/open-rec/rec-console) | Mode-aware operations UI for monitoring, diagnostics, Serving Graph, recall indexes, workflows, analytics, and models |
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
        Prometheus[(Prometheus)]
        Grafana[Grafana]
    end

    Operator[Operator] --> Console[rec-console<br/>standalone mode]

    API <-->|entities · behavior · filters| Redis
    API -->|recall · vector queries| ES
    API -->|recommendations| Web
    Prometheus -->|scrape API metrics| API
    Grafana -->|query metrics| Prometheus
    Source[Entity and event data] --> Algorithm[rec-algorithm<br/>local mode]
    Source -->|online entities and behavior| Redis
    Algorithm --> Dataset[Recall datasets]
    Dataset --> ES

    Console -->|entity diagnostics · Serving Graph| API
    Console -->|embedded monitoring dashboard| Grafana
```

Key characteristics:

- `rec-server` writes pushed data directly to Redis.
- `rec-algorithm` local mode turns entity and event data into recall datasets; the example imports
  source entities and behavior into Redis and recall datasets into Elasticsearch.
- Hot, new, i2i, and embedding recall data are queried from Elasticsearch.
- Redis retains online entities, behavior, blacklist, and exposure-filter state.
- `rec-console` provides monitoring, entity diagnostics, and Serving Graph management. Recall-index
  management, offline DAG, data analysis, Airflow automation, and Rank Model remain visible but are
  disabled as cluster-only modules.
- Prometheus collects rec-server metrics and Grafana provides the dashboard embedded by rec-console.
- Ranking runs in bypass mode; `rank-engine` is not required.
- Kafka, distributed processing, and Airflow are not required.

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
inference, observability, analytics, and control-plane operations.

```mermaid
flowchart TB
    Client[User · Web Demo · SDK] --> RecServer[rec-server<br/>cluster profile]
    Operator[Operator] --> Console[rec-console<br/>cluster mode]

    subgraph Online[Online recommendation path]
        direction TB
        RecallDAG[Serving DAG<br/>recall · filter · combine]
        RecallDAG --> Rank[rank-engine<br/>model inference]
        Rank --> Response[Ranked response]
    end

    subgraph Streaming[Real-time ingestion and feature path]
        direction TB
        Kafka[(Kafka)]
        Kafka --> Processor[Spark data-processor]
    end

    subgraph Storage[Serving and analytical storage]
        direction TB
        Redis[(Redis<br/>features · behavior · filters)]
        HBase[(HBase<br/>raw entity archive)]
        Hive[(Hive on HDFS<br/>partitioned entities and events)]
        ES[(Elasticsearch<br/>versioned recall indexes)]
        Models[(Versioned model artifacts)]
    end

    subgraph Offline[Scheduled and on-demand computation]
        direction TB
        Airflow[Airflow<br/>bootstrap · recall · rank DAGs]
        Runner[rec-algorithm runner]
        Spark[Spark cluster<br/>recall · rank · analytics jobs]
        Airflow -->|submit jobs| Runner
        Runner -->|spark-submit| Spark
    end

    subgraph Observability[Observability]
        direction LR
        Prometheus[(Prometheus)]
        Grafana[Grafana]
        Grafana -->|query metrics| Prometheus
    end

    RecServer -->|recommend request| RecallDAG
    Response --> Client
    RecServer -->|push API| Kafka

    RecallDAG -->|online state| Redis
    RecallDAG -->|active recall aliases| ES
    Processor -->|serving projection| Redis
    Processor -->|unaltered entities| HBase
    Processor -->|daily partitions| Hive

    Hive <-->|cumulative training and analysis data| Spark
    Spark -->|bulk-write staging index| ES
    Spark -->|train · evaluate · retain| Models
    Console -->|create · validate · switch · retain · rollback| ES
    Console <-->|DAG control · config · run state| Airflow
    Console -->|business analytics| Runner
    Console -->|entity diagnostics · Serving Graph| RecServer
    Console -->|activate · rollback model| Rank
    Models -->|load active version| Rank

    Prometheus -->|scrape API metrics| RecServer
    Console -->|embedded dashboard| Grafana
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

- Kafka and the Spark data-processor decouple online pushes from Redis projections, HBase raw
  entity retention, and immutable Hive/HDFS daily partitions.
- Airflow owns bootstrap, recall, rank-training, and rollback orchestration without managing Docker
  containers. The rec-algorithm runner turns those requests into Spark submissions.
- Spark reads cumulative Hive partitions for hot/new/i2i recall, rank training and evaluation, and
  on-demand business analytics.
- `rec-console` manages Airflow, recall-index releases, model activation and rollback, Serving Graph,
  entity diagnostics, analytics, and the embedded monitoring dashboard.
- `rank-engine` loads only evaluated model artifacts activated through rec-console.
- Prometheus collects rec-server API metrics and Grafana supplies the console monitoring dashboard.
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
| Infrastructure | Redis, Elasticsearch, Prometheus, and Grafana | Kafka, HDFS, Hive, HBase, Spark, Flink, Redis, Elasticsearch, Airflow, Prometheus, and Grafana |
| Push path | `rec-server` writes Redis directly | `rec-server` publishes to Kafka |
| Streaming processor | Not required | Spark data-processor |
| Offline scheduling | Not required | Airflow and Spark recall jobs |
| Recall data publishing | `rec-algorithm` local output and example import | Airflow, Spark, and `rec-console` version lifecycle |
| Ranking | Bypassed | `rank-engine` inference |
| Operations console | Monitoring, entity diagnostics, and Serving Graph | All rec-console modules |
| Online recall contract | Elasticsearch aliases and vector indexes | Same online contract |

Both modes use the same `rec-server` image, recommendation DAG, storage contracts, sample data,
and Web Demo. Runtime profiles change deployment behavior without changing the recommendation API.
