# Multiblockchain ELT - Phase 1

We present a prospective architecture doc for Multiblockchain ELT project - a blockhain data processing framework that is designed to be capable to provide application-specific data and analytics extracted from blockchain state changes at near-realtime speed and, where possible, horizontal scalability.

## The problem

Most blockchains are a distributed, fully-replicated key-value database with in-built rules for transaction processing. Due to a lot of pressure for transaction processing speed, data is stored in a way that would facilitate processing as much as possible, and is not really suitable for analytical queries or building user-facing applications at scale. The problem is especially pronounced for smart contract multitenant blockchains like Ethereum. Building apps over smart contracts almost always involves some kind of data-providing middleware that aggregates and indexes blockchain data, and that layer is often pretty complicated (e.g. TheGraph's subgraph of MakerDao is more than 1kloc). 

Even for simpler blockchains the task of making and maintaining an explorer, an analytical platform or a wallet is often tedious. Applications need a good way to have blockhain data sliced, diced, cleaned and enriched - and preferably in realtime. As a lot of those apps are financial, processing should be robust and error-free.

There can be multiple ways to approach this problem: most common is to interpret blockchain can be interpreped as a data stream - blocks, txs, events can be streamed and processed by an external system either in realtime or in batches. Other options are to build in indexing in the blockchain node itself (e.g. cosmos-sdk indexer, Filecoin Lotus), taking periodic snapshots of blockchain state via node API, or reinterpreting the underlying key-value database directly. Often app backends combine indexing solutions with direct queries to the node.

There's a number of advantages and gotchas to blockhain as data stream approach. Most important advantages are that:

- ordering and exactly-once delivery is trivial, blockchain can take care of that
- there already is a queriable consistent database that we can use for enrichment and integrity checking
- a lot of processing can be done in embarassingly parallel fashion

Constraints are more numerous:

- processing is often complex and follows closely the transaction procession logic of blockchain applications
- data structure is malleable: blockchains and smart contracts upgrade multiple times a day, need to be followed
- some applications require close to realtime porcessing and that requires low processing latency
- many blockhchains have forks and reorganizations that are pretty complex to deal with
- some processing can only be done in sequential fashion
- when missing a block or event or something we need a fast data backfill service

Those advantages and constraints directly influence the stack choice decision we decided to take and will describe in detail later.



## Related  work

For the purpose of our acrhitecture design we've analyzed blockchain indexing and analytics solutions, as well as industry grade even stream processing frameworks. The most rich ecosystem for blockchain data integration is Ethereum - it's got inbuilt indexing and rich GraphQL API in the node itself, multiple open- and closed-source ETL solutions (The Graph, EthereumETL, VulcanizeDB most prominent), and numerous analytics applications, like Dune Analytics and Bloxy. 

| Project        | Project Page                                                 | Source                                         | License          | Supported blockchain                         | Stack                           | API               | Latency          | Data Richness                  | Usage          | Most interesting features                                    |
| -------------- | ------------------------------------------------------------ | ---------------------------------------------- | ---------------- | -------------------------------------------- | ------------------------------- | ----------------- | ---------------- | ------------------------------ | -------------- | ------------------------------------------------------------ |
| Geth           | [https://geth.ethereum.org](https://geth.ethereum.org/)      | https://github.com/ethereum/go-ethereum        | LGPL-3.0 License | Ethereum and forks                           | Golang                          | JSON-RPC, GraphQL | immediate        | basic                          | 1000+ projects | Bloom filters for historical data, pretty good out of the box for low usage apps |
| Bitcore        | https://bitpay.com/blog/introducing-bitcore/                 | https://github.com/bitpay/bitcore/             | MIT License      | Bitcoin, Bitcoin forks, Ethereum (partially) | Typescript, Javascript, Mongodb | REST              | 10+ seconds      | historical                     | 30+ projects   | Very mature, full fledged block explorer                     |
| Cosmos SDK     | https://cosmos.network/sdk                                   | https://github.com/cosmos/cosmos-sdk           | Apache 2.0       | cosmos-sdk based                             | Golang                          | REST              | immediate        | basic                          | 30+ projects   | Granular indexing, extensible with custom indexers           |
| VulcanizeDB    | [https://vulcanize.io](https://vulcanize.io/)                | https://github.com/vulcanize/vulcanizedb       | AGPL-3.0 License | Ethereum                                     | Golang, PostgreSQL              | GraphQL           |                  | application-specific analytics | 2-5 projects   | Preferred tool for the MakerDAO and similar projects, less simple to extend than Graph Protocol which hampers adoption, can process storage diffs, not just events |
| EthereumETL    | https://twitter.com/ethereumetl                              | https://github.com/blockchain-etl/ethereum-etl | MIT License      | Ethereum                                     | Python                          | -                 | minutes to hours | historical                     | 2-5 prolects   | Internal Google project; ML analysis platform built on top of it |
| Dune Analytics | [https://www.duneanalytics.com](https://www.duneanalytics.com/) | -                                              | -                | Ethereum                                     | SQL                             | -                 | 90+ seconds      | application-specific analytics | 5+ projects    | SQL analytics and on the go visualizations makes for lightning fast adoption by the less-technical people; no API on top means that it's useless for DAPPs |
| Graph Protocol | [https://thegraph.com](https://thegraph.com/)                | https://github.com/graphprotocol/graph-node    | Apache 2.0       | Ethereum                                     | Rust, Typescript, PostgreSQL    | GraphQL           | 30+ seconds      | application-specific analytics | 5+ projects    | Soon to be decentralized; easy to extend and to build APIs on top and that drives adoption |
| Polkascan      | [https://polkascan.io](https://polkascan.io/)                | https://github.com/polkascan/polkascan-os      | GPL-3.0 License  | Substrate-based chains                       | Python                          | REST              | 10+ seconds      | historical                     | 2-5 projects   | Full-fledged block explorer; some sequential processing could be parallel; not easy to operate |
| Filecoin Lotus | [https://filecoin.io](https://filecoin.io/)                  | https://github.com/filecoin-project/lotus      | Apache 2.0       | Filecoin                                     | Golang, PostgreSQL              | -                 | immediate        | historical                     | 1 project      | Built-in PostgreSQL indexer                                  |

Below are a few inportan takeaways from Ethereum ecosystem data integration tools and beyond:

In-node indexing is helpful but doesn't cut it, there's just not enough analytical/querying features you can build upon what is a primarily an efficiency-constrained transactional engine over a key-value DB.

With hundreds of active, oft-updated blockchain applications a single team just can't follow those upgrades: application-specific processing and analytics should be easily extensible and offloaded to external teams. The system designes should facilitate the raw data acquisition and processed data storage as much as possible: that's the reason The Graph and Dune Analytics are so successful. First and foremost, buisiness logic should be easy to implement. 

The event intake should be as flexible as possible so that it wouldn't need to be upgraded every time the network upgrades. Preferably most of it should be offloaded to people who make API wrappers for respective blockchains.

Reorgs are hard to deal with and most data integration platforms rarely bother processing them. It's an advanced feature, a viable data integration platform can wait for finalization (e.g. Dune Analytics is currently a few minutes behind Ethereum latest block).

Despite that, near realtime analytics are important, especially for financial applications. A very successful 1inch dex aggregator, for example, has an ongoing struggle with TheGraph based APIs because it can often be a few blocks behind. 



### Event Stream Processing

The problem of just-in-time analytics over event streams is very well known in the larger world. it stems back to the 1990s, when first such systems were developed to process and analyze digital sensor data, stock trading, financial transaction for the purpose of fraud detection etc. It's rarely used to do what is essentially a repackaging of data from one database to another: that's usually purview of batch processing ETL systems like Apache Spark. Unfortunately, ETL systems are not optimized for low latency: there's no hope of using conventinal batch processing systems for under a minute latency. 

Nevertheless, event strram processors look like a tool for our job. Industry-grade frameworks are robust, scalable and battle-tested; the question is which ones are sutable for the job. There are three big categories: event stream processors (Apache Heron, Apache Storm, Apache Flink, Apache Spark, Apache Beam, Kafka Streams), Complex Event Processing (CEP) frameworks (Esper, Siddhi, etc), and specialized databases: ksqlDB, PipelineDB, etc. There is a great overview of the field in Baqend's blog, and a few research papers with coomparitive benchmarking, e.g. []. 



| Project                           | Project Page                                              | Source                                        | Class                               | License                     | Recommended infrastructure complexity | Estimated End-to-End Latency (90%) | Transformatio Options | Features                                                     | Known Drawbacks                                              |
| --------------------------------- | --------------------------------------------------------- | --------------------------------------------- | ----------------------------------- | --------------------------- | ------------------------------------- | ---------------------------------- | --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Apache Spark (microbatches)       | [http://spark.apache.org](http://spark.apache.org/)       | https://github.com/apache/spark               | Batch data processing framework     | Apache 2.0                  | Message queue + managed cluster + DB  | 3+s                                | SQL, JVM, Python, R   | Ubiquoutous usage, well-developed tooling                    | High latency                                                 |
| Apache Spark (Continuous Streams) | [http://spark.apache.org](http://spark.apache.org/)       | https://github.com/apache/spark               | Streaming data processing framework | Apache 2.0                  | Message queue + managed cluster + DB  | 200+ms                             | SQL, JVM, Python, R   | Very fast                                                    | No joins, little-used feature prone to misoperation          |
| Apache Storm                      | [https://storm.apache.org](https://storm.apache.org/)     | https://github.com/apache/storm               | Streaming data processing framework | Apache 2.0                  | Message queue + managed cluster + DB  | 1+s                                | SQL, Java             | Ubiquoutous usage, well-developed tooling                    | Requires an ops team                                         |
| Apache Heron                      | https://incubator.apache.org/clutch/heron.html            | https://github.com/apache/incubator-heron     | Streaming data processing framework | Apache 2.0                  | Message queue + managed cluster + DB  | 1+s                                | SQL, Java             | Apache Storm on steroids                                     | Requires an ops team                                         |
| Apache Flink                      | [https://flink.apache.org](https://flink.apache.org/)     | https://github.com/apache/flink               | Streaming data processing framework | Apache 2.0                  | Message queue + managed cluster + DB  | 300+ms                             | SQL, Java, Python     | Very fast                                                    | No JSON support in SQL yet, Requires an ops team             |
| Kafka Streams                     | [https://kafka.apache.org](https://kafka.apache.org/)     | https://github.com/apache/kafka               | Data stream processor               | Apache 2.0                  | Message queue + DB                    | 200+ms                             | JVM                   | Minimal dependencies                                         | Only for Java-based stacks                                   |
| ksqlDB                            | [https://ksqldb.io](https://ksqldb.io/)                   | https://github.com/confluentinc/ksql          | Streaming DB                        | Confluent License           | Message queue + DB                    | 200+ms                             | SQL, JVM              | Relatively simple infra, tested E2E latency (90%) was around 250ms | No good recovery strategy yet                                |
| ksql API workers                  | https://docs.confluent.io/4.1.0/ksql/docs/api.html        | https://github.com/confluentinc/ksql          | Data stream processor               | Confluent License           | Message queue + DB                    | 400+ms                             | Any stack             | Has bindings for basically any popular languages             |                                                              |
| pipelineDB                        | [https://www.pipelinedb.com](https://www.pipelinedb.com/) | https://github.com/pipelinedb/pipelinedb      | Streaming DB                        | Apache 2.0                  | DB                                    | 200+ms                             | SQL                   | Only requies PostgreSQL to work                              | Immature, ill-supported currently, limited functinality      |
| Siddhi                            | [https://siddhi.io](https://siddhi.io/)                   | https://github.com/siddhi-io/siddhi           | CEP framework                       | Apache 2.0                  | Message queue + DB                    | 400+ms                             | SQL-like DSL, Java    | Mature temporal logic patterns DSL                           | DSL instead of SQL, relatively high latency                  |
| Esper                             | http://www.espertech.com/esper                            | https://github.com/espertechinc/esper         | CEP framework                       | GPL v2                      | Message queue + DB                    | 400+ms                             | SQL, Java, .NET       | Mature temporal logic patterns DSL                           | Relatively high latency, hard to combine with unsupported stack |
| Materialize                       | [https://materialize.io](https://materialize.io/)         | https://github.com/materializeinc/materialize | Streaming DB                        | Business Source License 1.1 | DB                                    | 200+ms                             | SQL                   | Fast and rich streaming SQL processing, very promizing paradigm | Restricrive license, not fault-tolerant, inefficient scaling |

Apache family stream processors are good for what they were designed to do - namely, process billions of events in massive high availability deloyments. As such, they are not veyr practical to use if you don't have half a dozen engineers dedicated to support all the needed infrastructure for a suggested deployment - not very practical for our case! Flink might have been an exception to that, what with its ability to support Python processing functions, but it's stilly pretty hard to deploy and maintain, and doesn't have JSON realted functions in its native SQL processor yet.

Stream processing databases are, for the most part, too green to be relied upon, and pretty limited in fucntions. The exception to that is ksqlDB that is pretty mature and while still limited in processing features, is easily extensible with JVM-based UDF and any stack workers that use Kafka streams as sources and syncs. The downsides to ksqlDB are poor performance for analytics, lack of functional checkpointing mechanism for after-crash recovery and not a fully open-source license, but all three of those are irrelevant with our chosen overall design. 

In the end we decided to adopt Confuent stack: Kafka, Kafka Streams and ksqldb for processing with PostgreSQL sink for storage and analytics. Most of the processing should be done using ksqldb SQL requests, with possibly some user-defined functions for cleaning and trivial transformations (e.g. hex to SS58). When it's preferable to use non-Java processing, it can be set up as ksql or plain Kafka based side-workers. 

As much processing as possible should be done at streaming level; complex analytics requiring a lot of computations, complicated joins or purely sequential processing can be done on PostgreSQL data using SQL (views, materialized views), or external processors.

### Architecture

For the purpose of our ETL framework we consider the blockhain to be an event stream spitting out data in form of mostly freeform events, probably JSON-encoded. There is a global ordering over events but we can't guarantee them to be submitted in order, so we don't presume that. Events can be invalidated later in case of a reorg or a fork. 

Infrastructure-wise blockhcain data is streamed from a cluster of nodes behind a load balancer. Event listener for a blockchain is a simple service that subscibes to or polls blockchain updates, marshals them if needed and puts into processing queue. That part should be reliant on a best maintained API for a blockchain node, usually a JS API. Reliability and upgradeability are the most important parts of event listener.

We propose two stages of event processing: 

- collection stage - embarassingly parallel data cleaning and enrichment pipeline that can handle gaps and contradictions in blockhain related events
- sequential stage - complex processing that needs the data to be uniform, uncontradictory and in correct order

Both consists of multiple steps that clean, structure, slice, enrich and join data. Collection stage processing should deal mostly with local data and simple joins. As much processing as possible should be done in the collection stage, with a caveat that collection latency should remain sub-second for 99% percentile of events.

Sequential processing can include complex analytical queries, difficult joins, computation-heavy processing (e.g. ML calls), enrichment with external high-latency sources (e.g. price history APIs).  

Collection stage processing should be done, in order of preference, with:

- In-built SQL queries via ksqlDB
- Separate services using REST kSQL requests and Kafka to process things beyond what SQL can do
- SQL queries with UDFs via ksqlDB

It's better to avoid joins, enrichment with data via blockchain node API, or other external sources but it's possible.

Results of collection stage are posted to PostgreSQL sink database, and other sinks that want their data freshest (e.g. DeFi arbitrage bots). A watchdog process looks over to check for backfilling trigger (e.g. a block or two was missed and needs to be reprocessed), reorganizations and sequential processing watermark. When all the data up until block X had been collected, sequential processing for everything sequentialized but not yet processed is kicked off. 

We've built a prototype using this stack and end to end latency from block processed in node to collection finish looks to be under two second. We think that might be improved, and estimate sequencing stage for most of the blockchain applications to be sub-second. 





Articles and blogposts of interest



https://dpc.pw/rust-bitcoin-indexer-how-to-interact-with-a-blockchain

