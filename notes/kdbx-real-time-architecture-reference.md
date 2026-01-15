**Document Type**: Architecture Reference  
**Purpose**: Describe design patterns, trade-offs, and architectural options for real-time, event-driven KDB-X systems.  
**Non-Authoritative**: This document is not a specification and does not prescribe implementation choices.  
**Scope**: Used as a reference for deriving architecture decisions and implementation specifications.

# ABOUT DATA INTELLECT

Data Intellect is a consultancy specialising in financial and capital markets technology, with core expertise in:
- Event-driven and real-time systems
- Time-series data platforms
- High-performance software engineering
- Data analytics and solution delivery

The architectural patterns described in this document are derived from the design, implementation, and review of large-scale event-driven systems deployed within tier-1 financial institutions.


# INTRODUCTION

## Document Purpose and Structure

This document targets architects and engineers designing or reviewing event-driven, latency-sensitive systems using kdb+ or KDB-X.

KDB-X supports two primary use cases:
1. Time-series analytics across deep historical data through to real time
2. Real-time analytics and maintenance of derived data under latency constraints

This document focuses primarily on the second use case.

Topics covered include:
- Design principles for low-latency systems
- Performance-oriented architecture choices
- Resilience and observability considerations
- Common patterns for feed handling and data distribution

The document structure follows a typical real-time analytics data flow:
upstream data source → feed handler → distribution layer → real-time engine.

## KDB-X Is Not Just a Database

KDB-X combines two distinct capabilities:
- A high-performance time-series database
- A fully featured programming language (q)

Historically, q was designed as a programming language and later extended with database capabilities. As a result, systems built with KDB-X differ fundamentally from traditional database-centric architectures.

This flexibility enables rapid development of analytics and data pipelines, but it can also blur architectural boundaries.

In practice, there are trade-offs between:
- Implementing functionality directly in q
- Delegating responsibilities to external, specialized technologies

This document highlights common decision points where responsibilities may be better handled outside KDB-X, depending on latency, scalability, and operational constraints.

## Latency Targets

KDB-X is capable of supporting sub-millisecond processing for specialized workloads. However, its primary strength lies in efficiently executing vectorized analytics across large data sets, particularly as analytic complexity increases.

The q language enables analytics that are efficient in terms of:
- Developer productivity
- Execution time
- Sustained throughput

For the purposes of this document, the assumed target end-to-end latency range is:
- **1 to 200 milliseconds**

End-to-end latency is measured from the moment data enters the system to the point at which derived analytics become available to downstream consumers.

## Reference Architecture

The reference architecture assumed throughout this document follows a kdb+ tick-style, multi-process design.

A typical production deployment consists of at least the following components:
- Feed Handler: receives and normalizes data from upstream sources
- Tickerplant: logs incoming data and distributes events
- Real-Time Database (RDB): maintains in-memory state for real-time access
- Real-Time Engine (RTE): computes derived analytics and views

Production systems typically include additional supporting components, but these core elements form the foundation of most real-time KDB-X architectures.

From a concurrency perspective:
- Individual KDB-X processes are typically single-threaded
- Parallelism is achieved through multiple cooperating processes

The primary performance objectives in this architecture are:
- Minimizing end-to-end latency
- Maximizing sustainable throughput
- Achieving predictable and reliable behavior under load


## Disaster Recovery

Disaster recovery considerations are addressed within relevant sections of this document rather than as a standalone topic.

For latency-sensitive, highly available analytics platforms, a common approach is to:
- Deploy replicated environments
- Support controlled failover between environments in the event of significant failures

Specific recovery strategies depend on system topology, data durability requirements, and operational constraints.

## Hardware

Hardware configurations and operating system tuning are considered out-of-scope for this document. For up-to-date
benchmarks on optimal latency configurations refer to STAC.

## Code

It is assumed that the reader has some familiarity with q code. Code examples in this paper were created with the
Public Preview version of KDB-X. The code will also run using kdb+. Where possible example code is shared to
produce test data sets, the reader can modify these as required to align more closely to their own datasets.
As mentioned previously, computational timings will be presented as relative values. If required, readers should
conduct their own tests on their own hardware to measure absolute performance.

# DESIGN PRINCIPLES FOR A KDB-X SYSTEM

Building a successful data platform requires understanding who and what the downstream users and applications
are, what they care about and how they intend to use the data. Catering to all needs is challenging, but it is
achievable within a well architected system.

## Principles

The following design considerations commonly apply to KDB-X systems operating under latency constraints:

- **Latency and batching trade-offs**  
  Systems should be designed around the maximum tolerable latency. Introducing limited batching (e.g. publishing every few milliseconds) can significantly improve throughput and resource efficiency.

- **Event-driven processing**  
  Calculations are typically performed on data arrival rather than via periodic timers.

- **Separation of system and user processes**  
  Processes accessed by users should be isolated from those performing core system calculations to improve stability and resource control.

- **Single responsibility per process**  
  Each process should perform one clearly defined function.

- **Fail-fast behavior**  
  Processes should terminate quickly if they cannot initialize correctly due to configuration, environmental, or data issues.

- **Directed acyclic data flow**  
  Data flow is typically structured as a directed acyclic graph (DAG), avoiding feedback loops and bi-directional publishing.

- **Minimizing process chaining**  
  Long chains of dependent processes can complicate recovery and reduce resilience.

- **Efficient data types**  
  Appropriate data structures and datatypes are critical for performance.

- **Awareness of IPC costs**  
  Inter-process communication overhead should be considered explicitly in system design.

- **Minimizing unnecessary replication**  
  Where possible, derived analytics should be calculated once and published rather than recomputed independently.

- **Resilience and recoverability**  
  Systems should tolerate individual process failures and support independent recovery.

- **Appropriate technology selection**  
  Technology choices should consider performance, team expertise, operational complexity, and long-term maintainability. This may include using specialized messaging technologies or implementing components in languages such as C++, Java, or Python.

- **Observability as a first-class concern**  
  Monitoring, metrics, and operational visibility should be integral to system design.

## Trade-Offs

Some of these principles contradict one another. For example, minimising replication and ensuring tasks are discrete
may lead to more process chaining. These trade-offs are resolved and decided by engineering choices on a case-by-case basis.

## Implementing a Hot Path

Technology teams must balance optimizing for current business requirements with designing for future scale and flexibility.

A common architectural approach to support both highly latency-sensitive use cases and more general real-time analytics is to introduce a *hot path*.

A hot path is a dedicated data flow within the system where:
- Design choices are explicitly optimized for latency sensitivity
- Only a subset of data is processed
- Simplified data structures and datatypes are used
- Minimal transformation logic is applied
- Batching is reduced or eliminated

In some architectures, the hot path may enter the system via a separate tickerplant or distribution mechanism.

This approach can result in data duplication but enables strict latency optimization without constraining broader analytical or research use cases elsewhere in the system.


# FEED HANDLERS

## Introduction

The feed handler forms the boundary between a firm’s internal data systems and upstream data sources.

It connects to external or internal APIs and captures streaming data in its raw form. Upstream sources may include:
- External market data APIs
- Internal message transports
- Other real-time systems

At a minimum, the feed handler:
- Ingests raw events from the upstream source
- Structures data into tabular form
- Uses appropriate KDB-X datatypes for downstream ingestion and consumption


## Language Choice

Feed handlers must be implemented in a language that interfaces both with the upstream data source and with KDB-X.

Common high-performance language choices include:
- C++
- Java

Other languages such as Python or C# are possible but are less commonly used in latency-sensitive environments.

Feed handlers may also be implemented in q, often combined with embedded shared objects to manage upstream connectivity.

Language selection is influenced by:
- Performance requirements
- Consistency with the wider application codebase
- Internal standards
- Availability of appropriately skilled development resources


## Latency Choices

### Pruning, Enrichment, and Bespoke Logic

Not all data available on an upstream feed necessarily needs to be persisted in KDB-X.

In highly latency-sensitive systems, feed handlers typically prune unnecessary data as early as possible.

Feed handlers may also apply limited enrichment or bespoke business logic, such as:
- Symbology mapping to normalize identifiers across data sources
- Generation of derived datasets

For example:
- A feed handler may ingest full order flow data
- Maintain order book state
- Publish a derived top-of-book view alongside raw events

Transformation logic can alternatively be applied downstream of initial capture. This represents a design trade-off:
- Minimal transformation in the feed handler favors lower latency
- Additional transformation upstream can reduce repeated downstream processing and simplify consumption

The optimal approach depends on latency tolerance and system-wide efficiency considerations.


### Timestamping

It is common practice for feed handlers to apply a feed-time timestamp indicating when a message was received.

In some systems, multiple timestamps may be applied, for example:
- Time the raw message was received
- Time the processed event was published

Although timestamping introduces a small processing overhead, it provides valuable observability and ordering context for downstream components.

### Batching

Applying short batching intervals—either time-based or message-count-based—can significantly increase throughput for feed handlers and downstream components.

Batching improves:
- IPC efficiency
- Resource utilization
- Overall system throughput

Batch size and timing represent a trade-off between latency and throughput and must be chosen in line with system requirements.


### Conflation

Market data feeds often exhibit bursty or spiky message rates, particularly during periods of high market activity.

In latency-sensitive environments, conflation may be used to reduce message volume. For example:
- For top-of-book quote data, multiple updates for the same instrument may occur in rapid succession
- Only the most recent update may be propagated, discarding stale messages

Conflation reduces processing load and latency but results in incomplete datasets. This may negatively impact analytics or audit use cases that require full data fidelity.

The use of conflation must therefore be evaluated carefully against downstream requirements.


### Striping and Compute Resources

Feed handlers operating under steady-state conditions should maintain sufficient spare capacity to handle message rate spikes.

A common scaling approach is to *stripe* feed handlers, where multiple instances each process a subset of the data universe.

Striping dimensions may include:
- Instrument subsets
- Message types
- Topic or subscription groups

Striping decisions should consider downstream analytics requirements, particularly data ordering guarantees.

For example:
- Striping by instrument often preserves ordering across trades and quotes for a given instrument
- Striping by message type may complicate downstream analysis that relies on correlated event ordering


### Data Types and IPC

Data type selection and inter-process communication considerations apply equally to feed handler to KDB-X data transfers as they do to KDB-X internal communication.

Refer to the Performance Tuning section for detailed discussion of datatypes and IPC trade-offs.


## Resilience

Feed handlers may run on the same hosts as tick capture components or on separate hosts. Resilience considerations must therefore address failures at multiple levels, including:
- Individual processes
- Host machines
- Network connectivity
- Upstream data sources

In latency-sensitive analytics environments, resilience is commonly achieved through replication rather than complex in-process recovery logic.

A resilient KDB-X environment typically consists of replicated ingestion and distribution components operating in parallel (often in a hot–hot configuration). This approach mitigates the impact of single-point failures while maintaining continuous data availability.

Recovery strategies depend heavily on upstream data source capabilities:
- Some upstream sources support replay of missed data
- Others operate in a fire-and-forget mode, where previously published data cannot be recovered

Where replay is not available, replication is the primary mechanism for reducing the risk of permanent data loss.

More sophisticated resilience strategies are possible but introduce additional operational and architectural complexity. Selecting an appropriate approach is a design decision based on availability requirements, failure tolerance, and system complexity.


### Replicated Feed Handlers

A common resilience pattern is to deploy multiple independent instances of the same feed handler into replicated downstream environments.

Each feed handler:
- Connects independently to the upstream data source
- Operates without coordination with other feed handler instances
- Publishes data to its own downstream distribution layer

Where the upstream source does not provide replay or recovery capabilities, replication is often the only available method for mitigating data loss.

This pattern relies on the assumption that simultaneous failure of multiple independent ingestion paths is unlikely, thereby improving overall system availability.

#### Figure 3: Replicated Feed Handlers (Conceptual)

This diagram illustrates a resilience pattern in which multiple feed handlers ingest the same upstream data source and publish into separate, replicated downstream environments.

##### Components
- **Upstream Data Source**  
  Produces a real-time event stream. The source may or may not support replay of missed messages.

- **Feed Handler A**  
  Ingests data from the upstream source and publishes events to Tickerplant A.

- **Feed Handler B**  
  Independently ingests the same upstream data stream and publishes events to Tickerplant B.

- **Tickerplant A / Tickerplant B**  
  Separate tickerplant instances forming replicated downstream environments.

##### Data Flow
1. The upstream data source publishes real-time events.
2. Each feed handler independently receives events via separate connections.
3. Feed Handler A publishes events to Tickerplant A.
4. Feed Handler B publishes events to Tickerplant B.
5. Downstream consumers may subscribe to either replicated environment.

##### Resilience Characteristics
- Feed handlers and tickerplants operate independently.
- Failure of a single feed handler or tickerplant does not prevent ingestion into the alternate environment.
- The architecture reduces the impact of process, host, or network failures.

##### Recovery Considerations
- If the upstream source supports replay, data gaps may be recovered after reconnect.
- If the upstream source is fire-and-forget, replication is the primary mechanism for mitigating permanent data loss.
- This pattern does not imply coordination, deduplication, or ordering guarantees between replicated environments.

This architecture is conceptual and does not prescribe automated failover, data reconciliation, or consumer-side selection logic.


A variant of the above is depicted below, which might be the case if only one upstream connection is possible. In this,
one feed handler publishes to two tickerplant end points.

#### Figure 4: Single Feed Handler Publishing to Multiple Tickerplants (Conceptual)

This diagram illustrates a resilience variant used when only a single upstream connection is possible.

In this pattern, a single feed handler instance ingests data from the upstream source and publishes the same data stream to multiple downstream tickerplants.

##### Components
- **Upstream Data Source**  
  Provides a real-time data stream. Only a single connection is available.

- **Feed Handler A**  
  Maintains the sole connection to the upstream data source. Responsible for ingesting, normalizing, and publishing events.

- **Tickerplant A / Tickerplant B**  
  Separate tickerplant instances forming replicated downstream environments. Both receive data from the same feed handler.

##### Data Flow
1. The upstream data source publishes real-time events.
2. Feed Handler A receives the event stream via the single available connection.
3. Feed Handler A publishes each event to Tickerplant A.
4. Feed Handler A also publishes the same events to Tickerplant B.
5. Downstream consumers may subscribe to either replicated environment.

##### Resilience Characteristics
- Downstream distribution is replicated across multiple tickerplants.
- Failure of a single tickerplant does not prevent data ingestion into the alternate environment.
- The feed handler represents a single point of failure for upstream ingestion.

##### Recovery Considerations
- If Feed Handler A fails, ingestion from the upstream source stops entirely.
- Recovery depends on:
  - Restarting the feed handler
  - Upstream replay capabilities (if available)
- This pattern provides resilience at the distribution layer but not at the ingestion boundary.

##### Comparison with Fully Replicated Feed Handlers
- Compared to Figure 3, this architecture reduces upstream connection redundancy.
- It simplifies upstream connectivity but introduces a single ingestion failure domain.
- This trade-off is often necessary when upstream systems restrict the number of concurrent connections.

This architecture is conceptual and does not imply automatic failover, upstream replay, or deduplication between tickerplants.


#### Asynchronous Publishing and Downstream Connectivity

Feed handlers typically publish data to the distribution layer asynchronously in order to optimize throughput and minimize ingestion latency.

Asynchronous publishing introduces ambiguity in failure scenarios. If a feed handler becomes disconnected from the downstream distribution layer, it cannot reliably determine which messages were successfully received prior to disconnection. In particular, the last message sent by the feed handler may not be the last message persisted or propagated downstream.

When the upstream data source operates in a fire-and-forget mode and does not support replay, a common mitigation strategy is for the feed handler to maintain a local recovery log.

In this approach:
- The feed handler records normalized events locally before or alongside publishing
- Upon reconnection, the feed handler determines the last event successfully processed downstream
- Any missing events are re-published from the local recovery log

This recovery process may involve querying downstream components (e.g. an RDB) to identify recovery points.


#### Figure 5: Feed Handler with Local Recovery Log (Conceptual)

This diagram illustrates a recovery pattern used when a feed handler publishes asynchronously to a downstream distribution layer and the upstream data source does not support replay.

##### Components
- **Upstream Data Source**  
  Produces a real-time event stream. The source operates in a fire-and-forget mode and does not provide replay capability.

- **Feed Handler**  
  Ingests raw events from the upstream source, parses and normalizes the data, and publishes events asynchronously to the downstream distribution layer.

- **Local Recovery Log**  
  A persistent local log maintained by the feed handler. Records normalized events prior to or alongside publishing.

- **Distribution Layer (e.g. Tickerplant)**  
  Receives normalized events for logging and downstream propagation.

##### Data Flow
1. The upstream data source publishes raw events.
2. The feed handler receives raw events and performs parsing and normalization.
3. Normalized events are written to the local recovery log.
4. The feed handler publishes normalized events asynchronously to the distribution layer.
5. The distribution layer logs and propagates data to downstream consumers.

##### Resilience Characteristics
- Asynchronous publishing improves ingestion performance.
- Temporary disconnection between the feed handler and distribution layer does not immediately halt ingestion.
- The local recovery log provides a durable record of published intent.

##### Recovery Considerations
- If the feed handler disconnects from the distribution layer, it cannot assume which events were successfully received.
- Upon reconnection, the feed handler must determine the last event processed by the distribution layer (e.g. via downstream query).
- Events missing downstream are re-published from the local recovery log.
- This pattern is particularly important when upstream replay is unavailable.

This architecture is conceptual and does not prescribe specific log formats, retention policies, or reconciliation mechanisms.

### Impact on Downstream Consumers

Recovery actions performed by a feed handler may result in the publication of late or historical data into the system.

Downstream consumers must therefore be able to:
- Detect upstream ingestion failures
- Identify recovery or replay scenarios
- Avoid acting incorrectly on late-arriving data

This may require consumers to incorporate logic for:
- Timestamp validation
- Sequence or ordering checks
- Explicit handling of replayed events

Downstream handling of late data is a system-wide concern and must be considered as part of the overall architecture rather than treated as an isolated feed handler problem.


# DATA LOGGING AND DISTRIBUTION

## Introduction

Once data has been ingested from upstream sources, it must be distributed to downstream consumers.

In a standard KDB-X data capture architecture, this role is commonly performed by a tickerplant, although alternative distribution mechanisms are possible.

A performance-tuned tickerplant architecture can achieve low end-to-end latency, but it introduces several architectural considerations:
- All data flows through a single distribution component, creating a potential bottleneck
- The tickerplant represents a single point of failure unless replicated
- Slow or blocked consumers can cause backlogs and increased resource utilization

These characteristics must be considered when selecting and designing a distribution strategy.

## Standard Tickerplant

The tickerplant serves as both a persistence and distribution component within a KDB-X environment.

Its primary responsibilities are:
- Persisting incoming data to a durable log (analogous to a write-ahead log)
- Distributing data to subscribed downstream consumers

Downstream consumers may:
- Subscribe to real-time updates
- Read from the tickerplant log for recovery purposes, assuming shared or accessible storage

Tickerplants typically support:
- Immediate publishing, where updates are propagated as soon as they arrive
- Batched publishing, where updates are grouped and published on a timer
- Selective subscriptions by table or dataset

Because the tickerplant combines logging and distribution responsibilities, it can be considered to violate the “one job only” design principle. This trade-off is accepted in many architectures due to its simplicity and performance characteristics.


### Figure 6: Role of the Tickerplant in a KDB-X Environment (Conceptual)

This diagram illustrates the central role of the tickerplant as both a persistence and distribution component in a standard KDB-X data capture architecture.

#### Components
- **Feed Handler**  
  Ingests, parses, and normalizes data from upstream sources. Implemented in languages such as C++, Java, Python, or q.

- **Tickerplant**  
  Acts as the central ingestion point for normalized data. Responsible for both logging and distribution.

- **Tickerplant Log**  
  A persistent append-only log used for recovery and historical replay.

- **Real-Time Database (RDB)**  
  Maintains in-memory state for real-time analytics and query workloads.

- **Real-Time Engine (RTE)**  
  Consumes updates from the tickerplant to compute derived analytics and views.

#### Data Flow
1. The feed handler publishes normalized events to the tickerplant.
2. The tickerplant appends events to its persistent log.
3. The tickerplant distributes events to subscribed downstream components.
4. The RDB consumes events to update in-memory state.
5. The RTE consumes events to compute real-time analytics.

#### Functional Characteristics
- The tickerplant combines two responsibilities:
  - Durable logging of incoming events
  - Distribution of events to downstream consumers
- Publishing may occur immediately per event or in batches based on configuration.
- Consumers may subscribe selectively by table or dataset.

#### Architectural Implications
- The tickerplant represents a central throughput bottleneck, as all events pass through it.
- It is a single point of failure unless the environment is replicated.
- Slow or blocked consumers can increase resource pressure on the tickerplant.

This architecture is conceptual and does not prescribe replication, consumer throttling, or backpressure management strategies.


### Performance Considerations

The tickerplant can become a throughput bottleneck, as all ingested data flows through it for persistence and distribution.

To reduce pressure on the tickerplant, the following considerations commonly apply:

- **Storage performance**  
  Use high write-performance storage for the tickerplant log file to reduce I/O latency.

- **Consumer management**  
  Minimize the number of downstream consumers where possible and favor broadcast publishing to reduce per-consumer overhead.

- **Subscription filtering**  
  Custom filtered subscriptions reduce downstream data volume but increase load on the tickerplant, as it must generate filtered views dynamically.

- **Batching at the tickerplant**  
  Publishing data in small batches (e.g. using a short timer such as 10 ms) can significantly improve throughput.

- **Batching at the feed handler**  
  Batching updates before publishing to the tickerplant reduces the number of individual write operations, often eliminating the need for batching within the tickerplant itself.

- **Logging strategy**  
  Logging data only when it is published can increase throughput, but introduces additional resilience considerations.

When the primary objective is to minimize latency, a common initial configuration is:
- Tick-by-tick publishing from feed handlers
- Tick-by-tick publishing from the tickerplant
- A minimal number of downstream consumers
- Table-level subscriptions only (no additional filtering)
- Use of broadcast publishing


## Non-Logging Tickerplant

An alternative architecture is to decouple data distribution from data persistence by splitting the responsibilities of the tickerplant.

In this variant:
- The tickerplant is responsible only for distributing data
- Logging is performed by a separate, dedicated process

This approach aligns more closely with the “one job only” design principle and can reduce the latency of data distribution to downstream components.


### Figure 7: Non-Logging Tickerplant Variant (Conceptual)

This diagram illustrates a distribution architecture in which the responsibilities of logging and data distribution are separated.

#### Components
- **Feed Handler**  
  Ingests, parses, and normalizes data from upstream sources.

- **Tickerplant (Distribution Only)**  
  Receives normalized data and distributes it to downstream consumers without performing durable logging.

- **Logger Process**  
  Independently receives data from the tickerplant and persists it to durable storage.

- **Log File**  
  Persistent storage containing logged events.

- **Downstream Consumers**  
  Components such as RDBs and RTEs that subscribe to distributed updates.

#### Data Flow
1. The feed handler publishes normalized events to the tickerplant.
2. The tickerplant distributes events to downstream consumers.
3. The tickerplant also forwards events to the logger process.
4. The logger persists events to durable storage.

#### Performance Characteristics
- Distribution latency is reduced by removing logging responsibilities from the tickerplant.
- Logging and distribution can scale independently.

#### Recovery Considerations
- Recovery introduces additional complexity due to decoupled logging.
- A recovering downstream component must ensure that all prior events have been successfully persisted by the logger before replay or state reconstruction.
- This introduces a potential race condition between distribution and persistence that must be carefully managed.

This architecture is conceptual and does not prescribe synchronization, acknowledgment, or recovery coordination mechanisms.


## Reliable Transport

Reliable Transport (RT) is an alternative to the traditional tickerplant architecture, introduced through the KX kdb Insights SDK initiative.

In an RT-based system:
- Data publishers (e.g. feed handlers) write events to a local message log using the RT publishing API.
- Logged messages are pushed to an RT cluster for consumption and distribution.
- The RT cluster may operate across multiple nodes.

Compared to a standard tickerplant architecture, an RT-based approach typically exhibits the following characteristics:

- **Resilience**  
  Improved resilience when deployed as a multi-node cluster.

- **Consumer isolation**  
  Publishing, logging, and subscribing are decoupled, reducing the impact of slow consumers.

- **Scalability**  
  More easily scaled to higher message rates through clustering.

- **Latency trade-offs**  
  End-to-end latency is often higher than a performance-tuned tickerplant, though this is workload-dependent. For larger message sizes, latency may be comparable or lower.

Selection between a tickerplant-based and RT-based architecture depends on latency requirements, message rates, operational complexity, and resilience objectives.

## Dedicated Messaging Technology

An alternative architectural approach is to use a dedicated third-party messaging technology to propagate data throughout the system.

Such technologies may be used:
- For initial data distribution from feed handlers
- As a general-purpose message transport between system components

A common failure mode in growing KDB-X systems is excessive message fan-out performed directly by KDB-X components. Systems often begin with a small number of point-to-point TCP subscribers, but as the number of consumers grows, KDB-X processes may spend an increasing proportion of time performing IPC rather than analytics.

Dedicated messaging technologies address this by externalizing message distribution and fan-out.

Message transports span a range of architectures:
- **Broker-less systems** (e.g. Chronicle Queue, Aeron), which tend to offer lower latency
- **Broker-based systems** (e.g. Solace, Apache Kafka), which typically provide richer features and stronger observability

Most messaging systems:
- Support multiple producer and consumer technologies
- Enable cross-language interoperability
- Provide built-in recovery or replay mechanisms

Selecting a messaging technology involves trade-offs between latency, feature richness, operational complexity, and observability.

### Messaging Technology Examples

**Chronicle Queue**  
Chronicle Queue is commonly used in extremely latency-sensitive trading environments. It is optimized for high-throughput, low-latency message propagation, particularly within a single host. It relies on memory-mapped files to minimize copying and system calls.

**Solace**  
Solace targets general-purpose low-latency messaging and is available as both software and hardware appliances. The lowest latencies are typically achieved using hardware deployments. Solace supports:
- Hierarchical topic-based subscriptions with wildcards
- Message propagation across hosts, sites, and hybrid on-premise / cloud environments

Introducing a dedicated messaging layer expands architectural options for KDB-X systems and can improve observability. Fault tolerance and recovery responsibilities may be delegated to the messaging system rather than implemented directly within KDB-X components.

For example, downstream analytic components may rely on message replay features provided by the transport instead of recovering from a tickerplant log.

### Figure 10: Trading Environment with Central Message Bus (Conceptual)

This diagram illustrates a trading data architecture built around a central message bus that decouples data producers and consumers.

#### Components
- **Market Systems**  
  External sources providing real-time market data.

- **Feed Handler**  
  Ingests market data, performs parsing and normalization, and publishes events to the message bus.

- **Message Bus**  
  Central messaging layer responsible for data propagation, fan-out, and recovery.

- **KDB-X Environment**  
  Includes the following components:
  - **Tickerplant**: subscribes to the message bus and distributes data internally
  - **Real-Time Database (RDB)**: maintains in-memory state
  - **Historical Database (HDB)**: persists data for historical analysis
  - **Real-Time Engine (RTE)**: consumes data from the message bus and/or tickerplant to compute derived analytics

- **Pricing Engine**  
  External analytic component that both publishes to and subscribes from the message bus.

- **Trading Engine**  
  External trading logic that both publishes to and subscribes from the message bus.

- **Data Consumers**  
  Downstream systems and users consuming data products.

#### Data Flow
1. Market data is ingested by the feed handler.
2. Normalized data is published to the message bus.
3. The tickerplant subscribes to the message bus and distributes data internally to RDB and HDB.
4. The RTE both publishes derived analytics to and consumes inputs from the message bus.
5. Pricing and trading engines exchange data bidirectionally via the message bus.
6. Data consumers access historical data via the HDB and derived outputs from analytics components.

#### Architectural Characteristics
- The message bus decouples producers and consumers.
- Fan-out and recovery are delegated to the messaging layer.
- Multiple systems can publish and subscribe without direct coupling.

This architecture is conceptual and does not prescribe specific topic structures, replay semantics, or delivery guarantees.


# REAL TIME ENGINES

## When Should We Build RTEs?

A Real-Time Engine (RTE) consumes data from an upstream source, performs calculations or transformations, and makes results available to downstream consumers via queries or publication.

An RTE is typically appropriate when:
- Multiple users or systems require the same analytic output
- Analytics must be computed in an event-driven, low-latency manner
- Analytics are computationally expensive but require only incremental updates
- Intraday state must be maintained across incoming events

Compared to polling-based query approaches, event-driven RTEs are more complex to build and operate, but offer several advantages:
- A dedicated component focused on a specific analytic workload
- Data structures optimized for the task at hand
- No contention with shared query workloads
- Custom handling of multi-day or session-based state
- Potentially lightweight CPU and memory profiles

A common development approach is to first implement a polling-based solution to validate business value, then migrate to an event-driven RTE when justified.


## Language Choice

RTEs may be implemented in q or in any language with a supported KDB-X interface, such as Java, C++, C, C#, or Python (e.g. via PyKX).

In a standard KDB-X tick-style architecture, components typically written in q include:
- Tickerplant
- Real-Time Database (RDB)
- Historical Database (HDB)

For the purposes of this document, RTE implementations are discussed primarily in the context of q, which offers expressive syntax and strong performance characteristics for stateful, vectorized analytics.

## Performance

### Batching or Tick-by-Tick

The arrival pattern of upstream data strongly influences RTE design.

For latency-sensitive use cases:
- Tick-by-tick processing propagates each update immediately

For throughput-oriented use cases:
- Batch-based processing delivers multiple records per update

Batching improves throughput but introduces additional latency. The optimal approach depends on workload characteristics and latency requirements.


### Performance Characteristics

The resource utilization profile of an RTE depends on:
- Calculation complexity
- Volume of ingested data
- Cardinality of the data universe

RTEs commonly maintain state keyed by one or more identifiers. As the number of unique keys increases, memory requirements typically increase as well.

An efficient RTE implementation aims for:
- Memory usage that is constant with respect to total ingested volume, with linear growth relative to the number of unique keys
- CPU usage that scales sublinearly with ingested volume (due to batching)
- CPU usage that scales sublinearly with the number of unique keys where possible


## Example RTE: HLOC

This example describes an RTE that processes a stream of trade updates and maintains per-instrument:
- High
- Low
- Open
- Close
- Total traded volume

The RTE maintains state for each instrument and updates this state incrementally as new trades arrive.

On each incoming trade:
- **High** is updated to the maximum of the existing and incoming values
- **Low** is updated to the minimum of the existing and incoming values
- **Open** is set on first observation
- **Close** is updated on each new trade
- **Total volume** is incremented by the incoming trade size

To efficiently maintain state, the RTE should use data structures that provide constant-time lookup and update semantics. This typically involves a single keyed table, often with a `u` attribute applied to the key.

### Illustrative Implementation: Tick-by-Tick Processing

This example demonstrates a tick-by-tick update approach, where each incoming trade is processed individually.
It highlights the impact of key cardinality on performance.

/ Table to store HLOC state
hloc:([sym:`symbol$()]
  high:`float$();
  low:`float$();
  open:`float$();
  close:`float$();
  vol:`int$()
);

/ Generate example trade data
makedata:{[n;syms]
  ([] time:asc .z.p + n ? 0D00:00:01;
      sym:n?syms;
      price:n?100f;
      size:n?1000)
};

/ Process a single trade
updsingle:{
  current:hloc[x`sym];
  @[ `hloc; x`sym; :;
     ( x[`price] | current`high;
       x[`price] & 0Wf ^ current`low;
       x[`price] ^ current[`open];
       x[`price];
       x[`size] + 0 ^ current`vol )
  ];
};

### Performance Illustration: Key Cardinality

The following timings illustrate the effect of increasing the number of unique keys.
Timings are relative and hardware-dependent.

q)d1:makedata[100000;-10?`4]
q)\t updsingle each d1

q)d2:makedata[100000;-5000?`4]
q)\t updsingle each d2

### Optimisation via Unique Keys

Applying a `u` attribute to the symbol key removes the dependency on key cardinality by enforcing uniqueness.

q)delete from `hloc
q)update `u#sym from `hloc

q)\t updsingle each d1
q)\t updsingle each d2

### Illustrative Implementation: Batch Processing

This example extends the previous approach by processing batches of trades.
Batching reduces per-record overhead and enables sub-linear scaling characteristics.

/ HLOC table with unique key
hloc:([sym:`u#symbol$()]
  high:`float$();
  low:`float$();
  open:`float$();
  close:`float$();
  vol:`int$()
);

/ Batch update function
updjoin:{
  newdata:select
    newhigh:max price,
    newlow:min price,
    newopen:first price,
    close:last price,
    newvol:sum size
  by sym from x;

  newdata:newdata lj `sym xkey
    select sym, high, low, open, vol from hloc
    where sym in exec sym from newdata;

  `hloc upsert `sym xkey select
    sym,
    high:high|newhigh,
    low:newlow&0Wf^low,
    open:newopen^open,
    close,
    vol:newvol+0^vol
  from newdata;
};

### Performance Characteristics: Batch Processing

Relative timings demonstrate sub-linear scaling as batch sizes increase.

q)d1:makedata[100000;-10?`4]
q)\t updjoin d1

q)d2:makedata[100000;-5000?`4]
q)\t updjoin d2

q)d3:makedata[100;-5000?`4]
q)\ts:100 updjoin d3

q)d4:makedata[1000;-5000?`4]
q)\ts:100 updjoin d4

q)d5:makedata[10000;-5000?`4]
q)\ts:100 updjoin d5

q)d6:makedata[100000;-5000?`4]
q)\ts:100 updjoin d6

### Observations

- Tick-by-tick processing exhibits linear scaling with the number of updates
- Key cardinality significantly impacts performance without unique keys
- Applying a `u` attribute stabilizes lookup performance
- Batch processing provides orders-of-magnitude throughput improvements
- Execution time grows sub-linearly with batch size

### Notes

- Error handling, recovery, and concurrency concerns are omitted
- This example focuses on performance behavior rather than production robustness
- Absolute timings depend on hardware and configuration
- This implementation is illustrative and non-prescriptive


## Recovery

Real-Time Engines (RTEs) must be able to recover from intraday failures such that their post-recovery state is equivalent to the state that would have been reached had no failure occurred.

Recovery typically occurs during component initialization and must ensure that:
- All relevant historical data is processed exactly once
- No data is duplicated
- No data is missed

Recovery consists of two logically separate steps:
1. Processing historical data required to rebuild state
2. Subscribing to the live data stream

These steps must be carefully synchronized.  
Reading historical data before subscribing risks missing data.  
Subscribing before reading risks duplicating data.

Two commonly used recovery approaches address this synchronization challenge:
- Recovery using the Tickerplant log
- Recovery using the Real-Time Database (RDB)


### Recovery Using the Tickerplant Log

In a standard KDB-X architecture, the tickerplant subscribes to feed handlers, persists incoming events to a log file, and distributes data to downstream subscribers.

The primary purpose of the tickerplant log is recovery.

Typical characteristics include:
- Log files covering a defined time window (often a full trading day)
- End-of-day (EOD) log rotation
- Optional intraday rolling (e.g. hourly log files)

During startup or recovery, real-time subscriber processes (such as RDBs or RTEs) may use the tickerplant log to reconstruct state.


#### Figure 11: Recovery Using the Tickerplant Log (Conceptual)

This diagram illustrates a recovery sequence in which a real-time subscriber uses the tickerplant log to synchronize historical replay with live subscription.

##### Components
- **Real-Time Subscriber**  
  A process such as an RDB or RTE requiring recovery.

- **Tickerplant**  
  Distributes real-time data and maintains a persistent log.

- **Tickerplant Log**  
  Append-only log containing persisted events.

##### Recovery Sequence
1. The subscriber makes a synchronous request to the tickerplant to:
   - Subscribe to live data
   - Request metadata about the current tickerplant log file and the current message count (N)
2. The tickerplant returns:
   - The log file location
   - The current message count N
3. The subscriber replays events from the tickerplant log up to message N.
4. Once replay is complete, the subscriber continues processing live updates starting from message N+1.

##### Synchronization Guarantees
- The synchronous interaction with the single-threaded tickerplant ensures that no events are missed or duplicated between log replay and live subscription.

##### Trade-Offs and Limitations
- Subscriber processes must have access to the tickerplant log filesystem.
- Replaying large log files can be time-consuming, particularly later in the trading day.
- Replay activity may introduce backpressure on the tickerplant.
- Message counts are typically global rather than table-specific, which can be inefficient for subscribers requiring only a subset of data.

Frameworks such as TorQ introduce configurable log partitioning (e.g. per-table, per-hour logs) to mitigate some of these limitations.



### Recovery Using the RDB

Replaying the entire tickerplant log may be unnecessary or inefficient in some scenarios, particularly when:
- Only the latest state is required (e.g. building a cache of most recent prices)
- Only a subset of tables is relevant to the recovering process

In such cases, an RDB-oriented recovery approach may be preferable. The flow is outlined below.

#### Figure 12: Recovery Using the RDB (Conceptual)

This diagram illustrates a recovery pattern in which a real-time subscriber reconstructs state by querying the RDB rather than replaying the full tickerplant log.

##### Components
- **Real-Time Subscriber**  
  A process such as an RTE requiring partial recovery.

- **Tickerplant**  
  Provides live data and metadata about processed message counts.

- **Real-Time Database (RDB)**  
  Holds the current day's data in-memory.

##### Recovery Sequence
1. The subscriber synchronously subscribes to the tickerplant for only the required table(s).
2. The tickerplant returns the current processed message count (N) for the requested table(s).
3. The subscriber queries the RDB to retrieve historical data up to message N for those table(s).
4. The subscriber continues processing live updates received from the tickerplant.

##### Assumptions and Trade-Offs
- The RDB must hold all required data for the recovery window.
- This approach introduces an additional dependency on the RDB.
- Subscribers do not require access to the tickerplant log filesystem.
- Recovery is faster when only partial data is required.

For systems with intraday writes, recovery may involve a combination of RDB and HDB queries.



### Recovery Using On-Disk Cache Snapshots

Some RTEs maintain complex or long-lived state that may be expensive to reconstruct from raw event streams.

Examples include:
- Order book engines tracking Good-Til-Cancel orders across multiple days
- Analytics requiring multi-day aggregation

In such cases, periodically persisting a task-specific state snapshot to disk can significantly reduce recovery time.

During recovery:
- The RTE loads the most recent snapshot
- Only events that arrived after the snapshot was taken are replayed

Trade-offs include:
- Additional complexity in snapshot management
- Potential latency impact during snapshot persistence
- The need to ensure snapshot consistency with incoming data


### Downstream Recovery

When an RTE publishes derived data directly to downstream subscribers, it should support subscriber recovery mechanisms.

A common approach is to allow downstream consumers to:
- Request a snapshot of the current RTE state upon subscription
- Resume incremental updates from that snapshot

This enables downstream components to recover independently without requiring full replay of upstream data streams.



# PERFORMANCE TUNING

This section discusses lower-level considerations for optimizing latency-sensitive KDB-X systems.

The techniques described here are typically applied after higher-level architectural decisions have been made and obvious performance bottlenecks have been identified and addressed.

These approaches are intended to support fine-tuning and should be evaluated in the context of specific workload characteristics and performance objectives.


## Data Structuring and Datatypes

Data structure and datatype selection has a significant impact on memory usage, CPU efficiency, and overall system latency in KDB-X systems.


### Use of Symbols and Strings

In KDB-X, symbols are internally enumerated and stored in a global symbol table.

Symbols offer advantages in:
- Storage efficiency for repeating values
- Performance of operations such as filtering and grouping

However, unbounded growth of the symbol table can lead to performance degradation over time, particularly in systems that persist symbol data to disk.

For this reason:
- Symbols should be used carefully
- Strings are preferred for values that do not repeat frequently

A common example is order identifiers, which may repeat but typically not at a frequency that justifies symbol usage.

In real-time systems, symbols may be used more liberally for transient processing, provided they are not persisted to disk in that form and therefore do not pollute the historical database structure.

In some cases, alphanumeric values may be encoded into alternative datatypes for performance reasons. KDB-X provides utility functions for such encoding, and custom encoders may be implemented where appropriate.


### Nested Types

Nested data structures provide flexibility but may introduce additional memory overhead and reduced computational performance compared to flat, columnar representations.

KDB-X list storage characteristics include:
- A 16-byte header per list
- Allocation rounded up to the nearest power of two
- A minimum list size of 32 bytes

Nested lists introduce additional overhead through pointer indirection. Each nested element requires pointer storage, increasing memory consumption and reducing cache locality.


#### Figure 13: Memory Layout of Order Book Depth Data (Conceptual)

This figure contrasts two approaches to storing order book depth data:

##### Flat Columnar Representation
- Separate columns for each price and size level
- Each column consists of:
  - A header
  - A contiguous block of data values
  - Unused space due to power-of-two allocation

##### Nested Representation
- Prices and sizes stored as nested lists
- Each nested element contains:
  - A header
  - Data payload
  - Pointer indirection

##### Trade-Offs
- Nested structures offer flexibility for variable-depth order books
- Flat columnar structures offer significantly better memory efficiency and computational performance
- Fixed-depth representations may be acceptable for many analytic use cases


### Illustrative Code Example: Nested vs Flat Structures

The following example demonstrates memory usage and performance differences between nested and non-nested representations.
This example is illustrative and hardware-dependent.

// generate a dataset 
// in this case, 100000 rows and 3 price levels 
q)builddata:{[n;l] flip (`prices`sizes!(flip p;flip s)),(`$"ps"cross string 
1+tl)!(p:(tl:til[l])+\:.1*10+n?1000),s:(l,n)#(l*n)?1000} 
q)book:builddata[100000;3]
// sample of the table 
q)5 sublist book 
prices         sizes       p1   p2   p3   s1  s2  s3  ----------------------------------------------------- 
88.8 89.8 90.8 537 333 316 88.8 89.8 90.8 537 333 316 
55.4 56.4 57.4 861 459 21  55.4 56.4 57.4 861 459 21  
44.6 45.6 46.6 953 770 779 44.6 45.6 46.6 953 770 779 
24.5 25.5 26.5 592 430 338 24.5 25.5 26.5 592 430 338 
95.2 96.2 97.2 583 365 933 95.2 96.2 97.2 583 365 933 
// p1->p3 and s1->s3 columns contain 8 byte values 
// they each require 1MB of storage space  
// 100k 8 byte values, rounded up to the nearest power-of-2 
// therefore 6MB in total 
q)6*2 xexp ceiling 2 xlog 16+100000*8 
6291456f 
// size of each element of the price and size books in the nested case are 64 bytes 
q)2 xexp ceiling 2 xlog 16+3*8 
64f 
// nested list as a whole is (64*100000) + (size of list of pointers = 1048576) 
// double it for the two lists 
q)2*(64*100000) + 1048576f 
14897152f 
// nested lists in this instance are therefore twice the memory of non-nested 
// performance
// calculate the total quantity available in each book 
q)\ts:1000 select i,sum(s1;s2;s3) from book 
X 
// nested versions 
q)\ts:1000 select i,sum each sizes from book 
30X 
// can also do like this, though this is essentially 
// the same as converting back to the separated format 
q)\ts:1000 select i,sum flip sizes from book 
10X 
// what is the weighted average price for each book entry? 
q)\ts:100 select i, (s1;s2;s3) wavg (p1;p2;p3) from book  
Y 
q)\ts:100 select i,sizes wavg' prices from book  
23Y 
// assuming we wish to trade 1000 shares,  
// for each book entry, how many levels of the book do we have to trade through? 
q)\ts:100 select i, sum 1000>=sums(s1;s2;s3) from book 
Z 
q)\ts:100 select i, 1+(sums each sizes)bin' 1000 from book 
20Z 
// assuming we wish to trade 1000 shares 
// what VWAP price will we achieve through each book entry? 
q)\ts:100 select i, {[p;s;o] (deltas o&sums s) wavg p}[(p1;p2;p3);(s1;s2;s3);1000] from book 
A 
q)\ts select i, {[p;s;o](deltas o&sums s)wavg p}[;;1000]'[prices;sizes] from book 
100A

### Observations

- Nested structures consume significantly more memory than flat representations
- Pointer indirection reduces cache efficiency
- Flat, columnar representations are consistently faster for analytic operations
- Nested representations trade performance for flexibility

## Parallelisation via Secondary Threads

Secondary threads can be used to parallelize in-memory calculations.

In latency-sensitive environments, the benefit of parallelization depends on:
- Dataset size
- Computational complexity
- Overhead of thread management

For small datasets, the overhead of parallel execution may outweigh any benefit.

### Illustrative Code Example: Secondary Threads
This example demonstrates the impact of secondary threads on workloads of different sizes.

// extending the example above 
// generate two datasets 
// 100k rows, 3 price levels and  
// 100 rows, 3 price levels 
q)builddata:{[n;l] flip (`prices`sizes!(flip p;flip s)),(`$"ps"cross string 
1+tl)!(p:(tl:til[l])+\:.1*10+n?1000),s:(l,n)#(l*n)?1000}; 
q)book:builddata[100000;3]; 
q)smallbook:builddata[100;3]; 
// assuming we wish to trade different sizes  
// what VWAP price will we achieve through each book entry? 
// start with 0 secondary threads 
q)\s 0  
q)\t:100 exec i,{[p;s;o] (deltas o&s) wavg p}[(p1;p2;p3);sums (s1;s2;s3)] peach 500 1000 1500 2000 from book 
X 
q)\t:10000 exec i,{[p;s;o] (deltas o&s) wavg p}[(p1;p2;p3);sums (s1;s2;s3)] peach 500 1000 1500 2000 from smallbook 
Y 
// try with 4 threads 
q)\s 4 
// speed improvement for larger table 
q)\ts:100 exec i,{[p;s;o] (deltas o&s) wavg p}[(p1;p2;p3);sums (s1;s2;s3)] peach 500 1000 1500 2000 from book 
0.35X 
// but a slow down for the smaller table 
q)\ts:10000 exec i,{[p;s;o] (deltas o&s) wavg p}[(p1;p2;p3);sums (s1;s2;s3)] peach 500 1000 1500 2000 from smallbook 
2Y

### Observations

- Secondary threads improve performance for sufficiently large datasets
- For small datasets, thread management overhead can degrade performance
- Parallelization decisions must be evaluated in the context of workload size and latency sensitivity


## Inter-Process Communication

Inter-process communication (IPC) plays a significant role in the performance profile of latency-sensitive KDB-X systems.

Every message exchanged between KDB-X processes must be serialized before transmission and deserialized upon receipt. These operations consume CPU resources and can introduce non-trivial overhead, even when a process acts only as a pass-through relay.

### Serialization and Deserialization

Serialization and deserialization costs are often difficult to isolate in production systems, yet they contribute directly to end-to-end latency and CPU utilization.

Even when a KDB-X process does not modify a message payload, the message must still be fully deserialized on receipt and serialized again on transmission.

#### Illustrative Code Example: Serialization Cost

The following examples demonstrate the relative cost of serialization and deserialization for different datatypes.
Timings are relative and hardware-dependent.

// generate a dataset 
// in this case, a table containing 10 columns of 0f and 1000 rows 
q)builddata:{[n;c;d] flip (neg[c]?`4)!(c;n)#d}; 
q)floattable:builddata[10;1000;0f]; 
// time how long 1000 serialisations takes 
q)\t:1000 -8!floattable 
X 
// and de-serialisations 
q)r:-8!floattable 
q)count r 
91023 
q)\t:1000 -9!r 
3X 

The times are also impacted by the datatypes.

// symbols take longer, even though they produce a smaller result 
q)symtable:builddata[10;1000;`abc] 
q)\t:1000 -8!symtable 
3X 
q)count -8!symtable 
51023 
// strings longer still, and larger size 
q)stringtable:builddata[10;1000;enlist"abc"] 
q)\t:1000 -8!stringtable 
7X 
q)count -8!stringtable 
101023 


These serialisation and deserialiation times feed into overall data transfer times.

// open connection  
// all transfers are on the same host 
q)h:hopen 9999 
// measure transfer times 
q)\t:1000 h(set;`r1;floattable) 
Y 
q)\t:100 h(set;`r2;symtable) 
6Y 
q)\t:1000 h(set;`r3;stringtable) 
6Y 
// full transfer time only marginally more than serialise/deserialise times 
q)\t:1000 -9!-8!stringtable  
6Y 
// nested columns take longer than the equivalent wider table 
q)nestedints:builddata[10;1000;enlist 1 2 3i] 
q)\t:1000 -9!-8!nestedlongs 
Z 
q)wider:builddata[30;1000;1i] 
q)\t:1000 -9!-8!wider 
0.15Z 

#### Observations

- Serialization and deserialization costs can dominate IPC latency
- Symbols may serialize faster than strings but still incur overhead
- Nested structures significantly increase serialization time
- IPC overhead applies even to pass-through processes


### Compression and Encryption

Wire compression and encryption further increase serialization and deserialization costs.

- Encryption is disabled by default
- Compression is enabled by default for remote connections when message size exceeds 2 MB

These features improve security and bandwidth utilization but must be considered in latency-sensitive environments.

### Datatype Considerations

Datatype choices directly impact IPC performance.

To optimize data transfer:
- Avoid nested columns where possible
- Use flat, columnar structures
- Consider transmitting enumerated values (e.g. integers) instead of symbols
- Convert to symbols only at presentation or consumption boundaries

These optimizations increase development complexity and must be evaluated case by case.



### Asynchronous Broadcast

When the same message must be sent to multiple destinations, asynchronous broadcast can reduce serialization overhead.

Instead of serializing the message separately for each destination, the message is serialized once and broadcast to all recipients.

#### Illustrative Code Example: Async Broadcast

This example compares one-to-many messaging with and without async broadcast.


// build data set 
q)builddata:{[n;c;d] flip (neg[c]?`4)!(c;n)#d} 
q)stringtable:builddata[10;1000;enlist"abc"] 
// open 3 connections on localhost 
q)hs:hopen each 3#9999 
// timings for serialisation   
q)\t:1000 -8!stringtable 
X 
// timing to send async message to one client 
// is similar to serialisation time 
q)\t:1000 (neg hs[0])({x};stringtable) 
X 
// timing to send to 3 clients 
// approx 3x of sending to 1 
q)\t:1000 {x({x};stringtable)} each neg hs 
3X 
// timing to send to 3 clients using async broadcast 
// similar to sending to 1 
q)\t:1000 -25!(hs;({x};stringtable)) 
X

### Domain Sockets

Unix domain sockets may be used for local IPC on Unix-based deployments.

Compared to TCP sockets, domain sockets can provide improved performance for local process communication.


### Use of a Dedicated Message Transport

In latency-sensitive architectures, specialist messaging software may be used to offload responsibilities that could be handled within KDB-X but may not be optimal to do so.

A common example is high fan-out distribution with diverse subscription requirements.

See section 5.5 for further discussion.


### Changes in IPC

Future versions of KDB-X may introduce additional IPC configurability.

This document will be updated to reflect such changes as they become available.


## Memory

Memory behavior has a direct impact on latency predictability and system stability in KDB-X systems.


### Non-Constant Time Inserts

KDB-X uses a power-of-two allocation strategy to minimize frequent reallocations.

When a vector reaches its allocation limit and must grow, the resulting insert operation becomes expensive.

#### Illustrative Code Example: Insert Cost

This example demonstrates the cost of inserts when memory boundaries are crossed.


// create a dataset- each column in the table is a vector 
// align the size to a memory boundary 
q)builddata:{[n;c;d] flip (neg[c]?`4)!(c;n)#d} 
q)d:builddata[16777213;5;0f]                 
// first 2 inserts are cheap 
q)d:builddata[16777212;5;0f] 
q)\ts `d insert (1f;1f;1f;1f;1f) 
X 
q)\ts `d insert (1f;1f;1f;1f;1f) 
X 
// then expensive 
q)\ts `d insert (1f;1f;1f;1f;1f) 
>1000X  // much longer 
// then cheap again 
q)\ts `d insert (1f;1f;1f;1f;1f) 
X 


### Deletions

Deletions require memory reallocation and can be costly, particularly when removing small numbers of records.


// create a dataset- each column in the table is a vector 
q)d:builddata[17000000;5;0f] 
q)\ts delete from `d where i>16900000 
X 
// deleting one record is expensive 
q)\ts delete from `d where i>16899999 
0.4X 
q)\ts delete from `d where i>16899998 
0.4X 

### Garbage Collection

By default, KDB-X does not return freed memory to the operating system and instead recycles memory internally.

Garbage collection behavior can be controlled via the `-g` parameter:
- Deferred mode minimizes latency
- Immediate mode returns memory to the OS but increases overhead

The `.Q.gc[]` function may be used to explicitly trigger garbage collection.

#### Illustrative Code Example: GC Behavior

This example demonstrates the impact of garbage collection settings on performance and memory usage.


// create a dummy data set 
// 100m rows, universe of 100 syms 
q)makedata:{[n;syms] ([]time:asc .z.p+n?0D00:00:01; sym:n?syms; price:n?100f; size:n?1000)} 
q)d:makedata[100000000;-100?`4] 
// garbage collect, show memory usage 
q).Q.gc[] 
4294967296 
q).Q.w[] 
used| 4295395824 
heap| 4362076160 
peak| 8657043456 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 833 
symw| 40863 
// deferred garbage collect 
q)\g 0 
// time queries 
// test a couple of runs 
// subsequent runs are quicker due to memory already allocated 
q)\t select sum price, max size by sym from d 
X 
q)\t select sum price, max size by sym from d 
0.2X 
q)\t select sum price, max size by sym from d 
0.2X 
// show memory usage after – heap size has increased 
q).Q.w[] 
used| 4295396640 
heap| 5435817984 
peak| 8657043456 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 833 
symw| 40863 
// Change to immediate garbage collection 
q)\g 1 
q).Q.gc[] 
0 
// test queries 
// consistent overhead of returning memory to OS 
// Queries are slower than with deferred garbage collect 
q)\ts select sum price, max size by sym from d 
1.15X 
q)\ts select sum price, max size by sym from d 
0.3X 
q)\ts select sum price, max size by sym from d 
0.3X 
// resulting heap size is unchanged (memory returned to OS) 
q).Q.w[] 
used| 4295396640 
heap| 4362076160 
peak| 8657043456 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 833 
symw| 40863


### Memory Fragmentation

Memory fragmentation can prevent KDB-X from returning memory to the operating system, even when a large portion of memory is unused.

Fragmentation commonly arises from:
- Temporary creation of many small lists
- Non-uniform allocation and deallocation patterns

Reducing fragmentation requires careful memory usage patterns and may necessitate process restarts in long-running systems.


q).Q.w[] 
used| 427024 
heap| 67108864 
peak| 67108864 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 727 
symw| 37786 
q){@[`.;x;:;1000000?100]} each objects:`$'10#.Q.a 
`.`.`.`.`.`.`.`.`.`. 
q).Q.w[] 
used| 84313472 
heap| 134217728 
peak| 134217728 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 729 
symw| 37847 
q){eval(!;enlist`.;();0b;enlist enlist x)} each -1 _ objects 
`.`.`.`.`.`.`.`.`. 
q).Q.w[] 
used| 8815808 
heap| 134217728 
peak| 134217728 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 729 
symw| 37847 
q).Q.gc[] 
0 
q){eval(!;enlist`.;();0b;enlist enlist x)} last objects 
`.
q).Q.gc[] 
67108864 
q).Q.w[] 
used| 427200 
heap| 67108864 
peak| 134217728 
wmax| 0 
mmap| 0 
mphy| 17179869184 
syms| 730 
symw| 37877 


# Visualisation

Real-time analytics systems typically include visualisation components to allow users to observe, explore, and interact with data, as well as to support monitoring and operational observability.

KDB-X environments commonly leverage KX Dashboards for visualisation, although a range of alternative approaches and tools may be employed depending on requirements.


## Polling or Streaming

There are two primary data delivery models when building real-time visualisations:

- **Polling**: the user interface periodically requests data from backend components
- **Streaming**: backend components publish updates to the user interface as data changes

### Polling Model

A polling model is typically straightforward to implement. The UI issues queries against a KDB-X process at fixed intervals and renders the returned results.

Characteristics of a polling model include:
- Simpler development and deployment
- Predictable query patterns
- UI updates occur only at poll intervals

Trade-offs include:
- Increased server-side load due to repeated queries
- Potentially unnecessary network traffic
- Lack of true event-driven updates

With thin clients, polling is often limited to a single query per refresh cycle, requiring server-side joins when data originates from multiple sources. Thicker clients may join data locally, providing additional flexibility.

### Streaming Model

In a streaming model, backend components publish data updates to the UI as events occur.

Characteristics of a streaming model include:
- Event-driven updates
- Reduced redundant querying
- Improved scalability for larger user bases

Trade-offs include:
- Increased development complexity
- The need for server-side components to shape and manage streamed data
- Additional considerations around recovery and initial state

### Recovery Considerations for Streaming

Streaming visualisations often require an initial snapshot to provide historical context.

For example, a price chart displaying the last hour of data must retrieve historical data on initialization rather than building incrementally from the time the UI is opened.

As a result, streaming visualisations typically require:
- An initial snapshot or recovery query
- A transition from snapshot state to live updates

### Migration Pattern

A common approach is to:
1. Implement a polling-based visualisation for rapid prototyping
2. Validate business value and usability
3. Migrate to a streaming model when event-driven updates or wider rollout are required

For example, a polling query that retrieves the last hour of data every second is often simpler to implement initially than a fully event-driven streaming solution with recovery semantics.

### Update Frequency Considerations

Streaming updates intended solely for visualisation purposes do not typically require ultra-low latency.

A minimum update interval of approximately 200 milliseconds is generally sufficient, as this aligns with the threshold at which the human eye can perceive simple visual changes.



## Dashboards

Dashboarding tools allow users to rapidly build visualisations using pre-configured components.

Commonly used dashboarding solutions in KDB-X environments include:
- **KX Dashboards** (KX)
- **Pulse** (TimeStored)
- **Panopticon** (Altair)

These tools support both polling and streaming data delivery models and provide rich time-series visualisation capabilities.

### Strengths and Limitations

Dashboarding tools enable rapid prototyping and iterative exploration of ideas when combined with a productive KDB-X development environment.

They are particularly effective for:
- Early-stage experimentation
- Operational dashboards
- Standard analytic views

However, dashboarding tools may lack the fine-grained control required for highly customised or bespoke visualisations.

### Monitoring-Oriented Tools

Grafana is also commonly used in KDB-X environments, particularly for monitoring and support use cases rather than end-user analytics.

## Custom Builds

Custom visualisation applications can be built using any supported KDB-X interface and provide greater flexibility at the cost of increased development effort.

Custom builds are often initiated after prototyping in a dashboarding tool.

### Integration Options

KDB-X supports multiple integration paths for custom visualisation development:

- **Web-based UIs**  
  Browser-based applications using the KDB-X WebSocket interface, supporting both polling and streaming.

- **Python ecosystem**  
  Integration via PyKX (official), qPython (FINOS), or kola. Common tools include Plotly, Streamlit, and Jupyter.

- **Thick clients**  
  Applications built using C# or Java interfaces. Vendors such as 3Forge provide Java-based visualisation components integrated with KDB-X.

- **BI tools**  
  Connectivity via the Postgres SQL interface (pgwire), enabling standard BI tools to query KDB-X.

## Scaling

Visualisation components often begin with a small user base and later expand to wider groups.

If not carefully managed, this growth can:
- Increase query load on KDB-X components
- Increase IPC overhead
- Create fan-out bottlenecks similar to those described in section 5.5

### UI Caching Pattern

A common approach to scaling visualisation workloads is the introduction of a server-side UI cache.

The cache:
- Centralizes query execution
- Reduces duplicated backend requests
- Serves multiple clients efficiently


### Figure 14: KDB-X UI Cache (Conceptual)

This diagram illustrates a server-side caching layer used to support scalable visualisation.

#### Components
- **Real-Time Database (RDB)**  
  Provides raw or pre-aggregated data via timer-based queries.

- **Real-Time Engine (RTE)**  
  Publishes derived analytics to the cache.

- **UI Cache**  
  Maintains cached data suitable for visualisation and serves multiple clients.

- **Clients**  
  End-user applications consuming visualised data.

#### Data Flow
- The RDB executes timer-based queries to populate the UI cache.
- The RTE publishes derived analytics directly to the cache.
- Clients subscribe to or query the UI cache rather than backend components directly.

#### Benefits
- Reduced backend load
- Lower fan-out from KDB-X processes
- Improved scalability

### Fan-Out Considerations

High fan-out of visualisation data directly from KDB-X processes should generally be avoided.

For large user bases, consider introducing:
- Dedicated web servers
- Streaming gateways
- Message transports

These components can efficiently distribute data to many end users while isolating backend analytics processes from UI-driven load.


# MONITORING

## Introduction

# Monitoring

Monitoring is critical to the success of latency-sensitive analytics systems.

In such systems, monitoring serves two distinct purposes:
1. **Operational monitoring** (latency-insensitive), primarily acted upon by humans
2. **Programmatic monitoring** (latency-sensitive), reacted to automatically by system components

Both are important, but programmatic monitoring is particularly relevant in event-driven KDB-X architectures, where components may depend on upstream data or processes without direct coupling.


## Operational Monitoring

Operational monitoring focuses on system health, availability, and capacity planning.

It enables support teams to:
- Detect failures
- Identify degraded performance
- Understand resource utilization
- Plan future capacity

### Tooling and Metrics

Operational monitoring is typically implemented using standard observability platforms, including both on-premise and cloud-based solutions.

Common examples include:
- ITRS Geneos
- Datadog
- Dynatrace
- Prometheus

These platforms generally employ host-level agents that collect system and process metrics such as:
- CPU usage
- Memory consumption
- Disk I/O
- Network throughput

Metrics are aggregated centrally to provide a unified operational view. Log aggregation is also an important capability and may be provided by a separate tool.


### Application-Level Metrics

In addition to infrastructure metrics, application-specific metrics provide insight into the health of individual components.

In a KDB-X system, these may include:
- Table row counts and update rates
- Tickerplant queue lengths
- Process responsiveness
- Query execution counts and latency

Application metrics are typically collected centrally within the KDB-X environment and pushed to the monitoring platform by a dedicated process.

Best practice is to:
- Collect and publish raw metrics only
- Delegate alerting, thresholds, and anomaly detection to the monitoring platform
- Configure alerts based on historical behaviour rather than static thresholds

### Figure 15: Operational Monitoring (Conceptual)

This diagram illustrates a typical operational monitoring setup within a KDB-X environment.

#### Components
- **Tickerplant**
- **Real-Time Engine**
- **Process Monitor**
- **Monitoring Platform**

#### Data Flow
- The process monitor performs:
  - Queue health checks against the tickerplant
  - Data flow checks against the real-time engine
- The process monitor publishes metrics to the monitoring platform
- Alerting and status evaluation are performed outside KDB-X

This separation allows KDB-X components to remain lightweight and focused on analytics.

### Flow Check Nuances

Consider a metric designed to ensure data is flowing into a table at an acceptable rate.

Key challenges include:
- Data flow rates vary throughout the trading day
- A single table may receive data from multiple upstream sources

Most monitoring platforms support comparison against historical profiles that account for:
- Time of day
- Day of week
- Seasonal behaviour

To accurately detect failures, monitoring must be sufficiently granular to attribute flow issues to individual upstream sources rather than aggregated tables.



## Programmatic Monitoring

Programmatic monitoring ensures that analytic components act only on non-stale data.

Unlike operational flow checks, which are suitable for human response, programmatic monitoring must detect and react to failures quickly enough to prevent incorrect analytic output.

### Figure 16: Internal Dependency Chain (Conceptual)

This diagram illustrates a chain of analytic dependencies within a KDB-X system.

#### Components
- **Tickerplant**
- **Real-Time Position**
- **PnL Analytics**
- **Risk Analytics**

#### Dependencies
- Real-Time Position depends on execution data from the tickerplant
- PnL Analytics depends on:
  - Prices from the tickerplant
  - Positions from Real-Time Position
- Risk Analytics depends on PnL Analytics

As a result, Risk Analytics has indirect dependencies on:
- Execution data
- Price data
- Real-Time Position
- The tickerplant itself

### Status Propagation

If any upstream dependency becomes stale or unavailable, downstream analytics become invalid.

To manage this:
- Each component publishes its own status programmatically
- Downstream components subscribe to upstream status updates
- Staleness propagates through the dependency chain

For example:
- If Real-Time Position publishes a stale status due to an execution feed failure
- PnL Analytics should also mark its output as stale
- Risk Analytics should follow suit

### Granularity of Status

Status propagation does not need to be binary.

In some cases, partial staleness is required. For example:
- A subset of pricing instruments may become stale
- Only analytics dependent on that subset should be marked stale

Designing appropriate status granularity is use-case dependent and requires careful consideration.

### Process-Level Integration

Programmatic monitoring should integrate both data flow status and process status.

For example:
- A broken IPC connection triggering a `.z.pc` callback
- Should result in the publishing of a stale status
- Which then propagates downstream through the analytic chain

## Environment Consistency Monitoring

In replicated hot-hot deployments, both environments independently produce analytics.

Consistency monitoring verifies that:
- Both environments publish analytics for the same data
- Results are equivalent within an acceptable time window

### Trade-Offs

Environment consistency monitoring is difficult to act on programmatically.

Assuming no operational or programmatic alerts have fired:
- Differences usually require human investigation
- Root causes may include data discrepancies or race conditions

This type of monitoring primarily supports validation and assurance rather than automated remediation.

## Monitoring Subsystem

Monitoring data itself is time-series data.

KDB-X can be used to ingest, store, and analyse monitoring metrics alongside market and system data, enabling:
- Historical analysis
- Correlation between system behaviour and market activity
- Capacity and performance trend analysis


# Conclusion

Designing a performant, resilient real-time analytics system requires:
- A clear understanding of target objectives
- Sound architectural principles
- Awareness of how low-level design choices affect overall system behaviour

This document has outlined design patterns, trade-offs, and considerations derived from practical experience building event-driven KDB-X systems.

