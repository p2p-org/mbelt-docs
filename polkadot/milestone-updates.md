**Milestone 2 — Extraction and Load framework**

- We will create a framework to extract and store raw tx/block data from Polkadot Relay chain and Kusama, suitable both for blockchains with finality gadgets and reorganizible chains
- We will develop a reference implementation of the said pipeline for Kusama and Polkadot, including
  - an event listener for substrate-based chains using polkadot-js and websocket subscriptions API of a substrate node that will work on Kusama and Polkadot
  - event cleaning and enrichment transformation based on ksqlDB and ksql REST API
  - PostgreSQL sink for cleaned and enriched data
  - Backfill watchdog and data integrity watermark services

- We will provide documentation and tutorials on running the extraction pipelines and making a custom extraction pipeline

- Documentation will be delivered according to the [Web3 Milestone Deliverables Guidelines](https://github.com/w3f/Web3-collaboration/blob/master/grants/milestone-deliverables-guidelines.md).

**Milestone 3 — Enrichment, API endpoints, Documentation**

- We will create enrichment transformations for the sequenced blockchain chain data, including specialized structured data warehouses for transfers, staking, governance on Polkadot and Kusama based on ksqlDB, ksql REST API, PostgreSQL views and materialized views
- We will create a GraphQL API over raw and enriched data extracted from the chain
- We will provide documentation and tutorials on running the pipelines we made, hosting an API, using the API and developing your own pipelines and APIs for other blockchains.
- We will provide a docker-compose setup for easy deployment of the ETL and API solution
- Documentation will be delivered according to the [Web3 Milestone Deliverables Guidelines](https://github.com/w3f/Web3-collaboration/blob/master/grants/milestone-deliverables-guidelines.md).
