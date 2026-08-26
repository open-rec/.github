# OpenRec

**An end-to-end open-source recommendation platform that grows from a single machine to a
distributed data stack without changing your application API.**

OpenRec connects data ingestion, streaming and batch processing, recall generation, configurable
online serving, model training, experimentation, observability, and operations in one system. Use
the lightweight standalone distribution for development and small-to-medium workloads, or enable
the cluster distribution for Kafka, Spark, Airflow, versioned recall indexes, and model lifecycle
management.

[Get started](https://github.com/open-rec/example#quick-start) ·
[Standalone guide](https://github.com/open-rec/example/tree/master/example_standalone) ·
[Cluster guide](https://github.com/open-rec/example/tree/master/example_cluster) ·
[Architecture](https://github.com/open-rec/example/blob/master/docs/architecture.md) ·
[Contributing](https://github.com/open-rec/.github/blob/master/CONTRIBUTING.md) ·
[Security](https://github.com/open-rec/.github/blob/master/SECURITY.md)

## Why OpenRec

- **One API, two deployment modes.** Start locally, then move ingestion, features, recall, and
  ranking onto distributed infrastructure without replacing the recommendation contract.
- **Complete recommendation lifecycle.** Ingest users, items, and events; compute recall tables;
  train and publish models; serve recommendations; measure and operate the system.
- **Configurable online DAG.** Compose recall, filtering, ranking, combination, and operation nodes,
  then publish or roll back graph snapshots through the control plane.
- **Safe artifact lifecycle.** Recall indexes and rank models are immutable, validated, activated
  atomically, retained by policy, and explicitly rollbackable.
- **Observable by design.** Request metrics, dependency health, business analytics, entity
  diagnostics, Airflow state, and Grafana dashboards are available from the operations console.
- **Replaceable components.** Storage, recall algorithms, operation rules, processing engines, and
  clients are separated into focused repositories with documented contracts.

## Quick start

Clone the distribution repository together with its sibling components, then start the smallest
complete recommendation chain:

```shell
git clone https://github.com/open-rec/example.git
cd example
./scripts/checkout-components.sh
./example_standalone/start.sh
```

After the acceptance check succeeds, open the Web Demo at `http://127.0.0.1:12345` and the OpenRec
Console at `http://127.0.0.1:8095`.

> The distribution is under active development. Review the deployment guide and replace all sample
> credentials before exposing any service outside a trusted development network.

## See OpenRec in action

The Web Demo exercises the same recommendation API used by applications and SDKs. Switch users,
scenes, and experiments; inspect recall and rank scores; then send exposure, click, collect,
purchase, or negative-feedback events to see the next recommendation respond to behavior.

![OpenRec Web Demo showing personalized recommendations, recall channels, ranking scores, and feedback controls](images/rec_web_demo.png)

The OpenRec Console brings the serving and data planes into one control surface. Operators can
inspect and publish recommendation graphs, govern immutable recall and model versions, configure
offline workflows, and follow production traffic without assembling separate administration
tools.

### Serving Graph and experiments

Visually inspect the live recommendation DAG, tune node configuration, and maintain experiment
variants with an explicit default fallback. The graph view makes recall fan-out, filtering,
combination, ranking, operation rules, and collection paths visible together.

![OpenRec Console Serving Graph editor showing an online recommendation DAG and node configuration](images/ab_console.png)

### Artifact and workflow lifecycle

| Recall index operations | Offline recommendation workflow |
|---|---|
| ![Recall index versions with active and standby states and rollback controls](images/recall_console.png) | ![Offline Airflow DAG tasks and recommendation workflow configuration](images/offline_console.png) |
| Review hot, new, and i2i index versions, atomically switch the active alias, retain rollback candidates, and restore the previous version. | Inspect task dependencies and publish configuration for scheduled recall pipelines managed through Airflow. |

### Ranking and observability

| Rank model lifecycle | Monitoring dashboard |
|---|---|
| ![Rank model training, evaluation, release, retention, and rollback controls](images/rank_console.png) | ![Embedded Grafana dashboard showing recommendation API traffic and latency](images/monitor_console.png) |
| Train LR or FM candidates, enforce evaluation gates, publish an approved model, and roll back to a retained version by scene. | Track push and recommendation QPS, latency, errors, result volume, runtime components, and experiment dimensions in the embedded Grafana view. |

Recall releases, offline workflows, rank-model lifecycle, and full experiment operations are
cluster capabilities. Standalone mode keeps the same Web Demo and provides monitoring, entity
diagnostics, and Serving Graph management for local development and integration testing.

## Product landscape

```mermaid
flowchart LR
    Client[Applications and SDKs] --> Serving[rec-server<br/>online serving DAG]
    Operator[Operators] --> Console[rec-console<br/>control plane]

    subgraph DataPlane[Data and model plane]
        Stream[data-processor<br/>streaming projections]
        Offline[rec-algorithm<br/>recall · feature · rank]
        Rank[rank-engine<br/>model inference]
        Stores[(Redis · Elasticsearch<br/>Kafka · HDFS · Hive · HBase)]
    end

    Serving --> Rank
    Serving <--> Stores
    Stream --> Stores
    Offline --> Stores
    Console --> Serving
    Console --> Rank
    Console --> Offline

    Platform[bigdata-platform<br/>infrastructure · scheduling · monitoring] --- Stores
    Distribution[example<br/>release · deployment · end-to-end CI] --- Serving
    Distribution --- Console
```

### Recommendation lifecycle

```mermaid
flowchart LR
    Ingest[Users · items · events] --> Project[Online projection]
    Ingest --> Lake[Historical data]
    Lake --> Compute[Recall and model jobs]
    Compute --> Validate[Validate immutable artifacts]
    Validate --> Activate[Atomic activation]
    Activate --> Recommend[Online recommendation DAG]
    Recommend --> Feedback[Exposure · click · conversion]
    Feedback --> Ingest
    Observe[Metrics · analytics · experiments] --> Compute
    Observe --> Recommend
```

## Deployment modes

| Capability | Standalone | Cluster |
|---|---|---|
| Intended use | Local development and small-to-medium integration | Distributed, production-oriented integration |
| Application API and serving DAG | `rec-server` | The same `rec-server` contract |
| Online state | Redis | Redis projections from Kafka |
| Recall and vector indexes | Elasticsearch | Versioned Elasticsearch indexes and active aliases |
| Ingestion | Direct write | Kafka mutation envelope |
| Processing | Local import and algorithms | Spark/Flink streaming and batch processing |
| Scheduling | Manual/local | Airflow |
| Ranking | Optional bypass | Versioned `rank-engine` models |
| Operations | Monitoring, entities, serving graph | Full workflow, release, analytics, model, and experiment control |
| Observability | Prometheus and Grafana | Prometheus and Grafana plus distributed service metrics |

Both modes use the same sample data, recommendation API, serving graph contract, and Web Demo. The
deployment profile changes the data path rather than the application integration.

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

## Component map

| Repository | Responsibility | Start here when… |
|---|---|---|
| **[example](https://github.com/open-rec/example)** | OpenRec distribution, deployment guides, version manifest, runnable examples, and end-to-end CI | You want to install, evaluate, or release OpenRec |
| **[rec-server](https://github.com/open-rec/rec-server)** | Java online service, recommendation protocol, serving DAG, recall/filter/rank nodes | You are extending online recommendation behavior |
| **[rec-console](https://github.com/open-rec/rec-console)** | Operations UI and APIs for graph, workflow, recall, model, experiment, analytics, and diagnostics | You are operating or governing the platform |
| **[rec-algorithm](https://github.com/open-rec/rec-algorithm)** | Local and Spark recall, embedding, feature, ranking, evaluation, and publication jobs | You are developing offline algorithms |
| **[rank-engine](https://github.com/open-rec/rank-engine)** | Rank-model training/release support and online inference | You are adding or serving ranking models |
| **[data-processor](https://github.com/open-rec/data-processor)** | Kafka mutation processing and Redis/HBase/Hive projections with Spark and Flink | You are changing the real-time data path |
| **[bigdata-platform](https://github.com/open-rec/bigdata-platform)** | Containerized storage, messaging, compute, scheduling, and monitoring infrastructure | You are deploying or debugging dependencies |
| **[sdk](https://github.com/open-rec/sdk)** | Application client SDKs | You are integrating an application |
| **[model](https://github.com/open-rec/model)** | Sample recall datasets, features, and rank artifacts | You need reproducible sample artifacts |

Dependency direction and repository ownership are maintained in the
[distribution architecture](https://github.com/open-rec/example/blob/master/docs/architecture.md).

## Quality and releases

The [example distribution](https://github.com/open-rec/example) is the compatibility authority for
a complete OpenRec version. Its manifest records the component refs used by a release, and its CI
verifies four levels:

1. Repository policy, shell, Python DAG, Compose, and manifest validation.
2. Cross-repository Java compilation and focused unit tests.
3. Standalone startup, sample import, serving-graph execution, and a real recommendation request.
4. Cluster ingestion, streaming projection, Hive persistence, recall publication, model lifecycle,
   analytics, deletion semantics, and online serving.

Component repositories remain independently testable and releasable, while the distribution
defines which versions are known to work together.

## Community

OpenRec welcomes bug reports, documentation improvements, integrations, algorithms, SDKs, and
production feedback.

- Read the [contribution guide](https://github.com/open-rec/.github/blob/master/CONTRIBUTING.md)
  before opening a pull request.
- Use the relevant component repository for scoped bugs and changes; use
  [example issues](https://github.com/open-rec/example/issues) for installation, release, or
  cross-component problems.
- Follow the [Code of Conduct](https://github.com/open-rec/.github/blob/master/CODE_OF_CONDUCT.md).
- Report vulnerabilities privately according to the
  [security policy](https://github.com/open-rec/.github/blob/master/SECURITY.md).
- See [Support](https://github.com/open-rec/.github/blob/master/SUPPORT.md) for help routing and the
  information required for effective diagnosis.

## License

OpenRec components are released under the Apache License 2.0 unless a repository states otherwise.
Third-party services and dependencies retain their respective licenses.
