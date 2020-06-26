# Multiblockchain ELT - Phase 1

We present a prospective architecture doc for Multiblockchain ELT project - a blockhain data processing framework that is designed to be capable to provide application-specific data and analytics extracted from blockchain state changes at near-realtime speed and, where possible, horizontal scalability.

## The problem

Most blockchains are a distributed, fully-replicated key-value database with in-built rules for transaction processing. Due to a lot of pressure for transaction processing speed, data is stored in a way that would facilitate processing as much as possible, and is not really suitable for analytical queries or building user-facing applications at scale. The problem is especially pronounced for smart contract multitenan blockchains like Ethereum. Building apps over smart contracts almost always involves some kind of data-providing middleware that aggregates and indexes blockchain data, and that layer is often pretty complicated (e.g. TheGraph's subgraph of MakerDao is more than 1kloc). 

Even for simpler blockchains the task of making and maintaining an explorer, an analytical platform or a wallet is often tedious. Applications need a good way to have blockhain data sliced, diced, cleaned and enriched - and preferably in realtime. As a lot of those apps are financial, processing should be robust and error-free.

There can be multiple ways to approach this problem: most common is to interpret blockchain can be interpreped as a data stream - blocks, txs, events can be streamed and processed by an external system either in realtime or in batches. Other options are to build in indexing in the blockchain node itself (e.g. cosmos-sdk indexer, Filecoin Lotus), taking periodic snapshots of blockchain state via node API, or reinterpreting the underlying key-value database directly. 

There's a number of advantages and gotchas to blockhain as datastream approach. Most important advantages are that:

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

Blockchains: EthereumETL, TheGraph, Dune Analytics, Bloxy, Polkascan/Subscan, ...

Fortunately, the problem of just-in-time analytics of event streams. There are multiple projects made just for that purpose, and many of them are open source. There are three big categories: event stream processors (Apache Heron, Apache Storm, Apache Flink, Apache Spark, Apache Beam, Kafka Streams, ...), Complex Event Processing (CEP) frameworks (Esper, ...), and specialized databases: ksqlDB, PipelineDB, Clickhouse, ..



Most of Apache Family stream processors are good for what they were designed to do - namely, process billions of events in huge high availability deloyments. As such, they are not veyr practical to use if you don't have half a dozen engineers dedicated to support all the needed infrastructure for a standard deployment. Also, they are almost always purely Java-centric and are not easily used with any other stack. Flink might have been an exception to that, what with its ability to support Python processing functions, but it's stilly pretty hard o deploy and maintain, and doesn't have JSON realted functions in its native SQL processor yet.

CEP frameworks we decided are not really useful for our purpose because they're all about stateful computation on data stream to reduce the load on data sink - transforming trillions of messages from, say, IOT gauges to megabytes of actionable data in the database. For our puropose we'd like to stick to mostly stateless processing where possible, and we won't really do data reduction. 

Stream processing databases are, for the most part, too green to be relied upon, and pretty limited in fucntions. We decided to adopt Confuent stack: Kafka, Kafka Streams and ksqldb for processing with PostgreSQL sink for storage and analytics. Most of the processing should be done using ksqldb SQL requests, with possibly some user-defined functions for cleaning and trivial transformations (e.g. hex to SS58); complex tasks can be done as Kafka Streams plugs. When it's preferable to use non-Java processing, it can be set up as ksql or plain Kafka based side-workers. 

As much processing as possible should be done at streaming level; complex analytics requiring a lot of computations, complicated joins or purely sequential processing should be done 

Architecture

Blockchain is treated primarily as data stream

There are two stages of processing: 

- embarassingly parallel (collection) stage
- sequential (complex) stage

Both consists of multiple steps that clean, structure, slice, enrich and join data. Collection stage processing should deal mostly with local data and simple joins. As much processing as possible should be done in the collection stage, but collection latency should remain sub-second for 99% percentile.

Sequential processing can include complex analytical queries, difficult joins, computation-heavy processing (e.g. ML calls), enrichment with external high-latency sources (e.g. price history APIs). 

Collection stage processing can be done, in order of preference, with:

- Vanilla SQL queries via ksqlDB
-  Separate services using REST kSQL requests and Kafka to process things beyond what SQL can do
-  SQL queries with UDFs via ksqlDB

It's better to avoid joins, enrichment with data via blockchain node API, or other external sources. 

Results of collection stage are posted to PostgreSQL sink database, and other sinks that want their data freshest (e.g. DeFi arbitrage bots). A few watchdogs look over to check for backfilling trigger (e.g. a block or two was missed), reorganizations and sequential processing watermark. 

When all the data up until block X had been collected, sequential processing for everything up to this block is kicked off. 

We've built a prototype using this stack and end to end latency from block processed in node to collection finish looks to be under two second. We think that might be improved, and estimate sequencing stage for most of the applications to be sub-second. 



