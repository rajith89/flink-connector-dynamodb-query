# Flink Driver for DynamoDB (SQL Batch Read)

We need a JDBC driver to be used in Apache Flink 1.17 to read (i.e. query) data from a DynamoDB table to be used in Flink SQL.

## Requirements
* It must work with Apache Flink 1.17 in a Flink SQL job:
  ```sql
    CREATE TABLE `source_Swap` (
        `entityId` STRING,
        `entityUpdatedAt` BIGINT,
        `chainId` INT,
        `contractAddress` STRING,
        PRIMARY KEY (`entityId`) NOT ENFORCED
    ) WITH (
        'connector' = 'flair-dynamodb',
        'namespace' = 'sushiswap',
        'entity-type' = 'Swap',
        
        # Optionally users might want to partition the scan.
        # This is handled by the Flink core and it'll provide these values to the connector
        # and then connector can decide what to do with these values. They can be used to construct the ddb Query.
        'scan.partition.num' = '10',
        'scan.partition.column' = 'blockNumber',
        'scan.partition.lower-bound' = '{{ chrono("61 days ago") }}',
        'scan.partition.upper-bound' = '{{ chrono("now") }}'
    );
  ```
* We mainly need the driver to work for a specific structure of DynamoDB table and primary key. Table name needs to be configurable via SQL props (the default table name is `g-graph_simple-entities`).
* It must query all the rows in that table based on the configured namespace + entity-type (using proper index) with correct pagination.
* There are some special "internal" columns on DynamoDB that must not be exposed, or must be renamed before providing to the user (read the "Virtual fields" section below).
* We will give you access to some existing code for inspiration and quick iteration:
  - Existing JDBC driver for Rockset (which is basically very similar functionally except it's reading from Rockset vs DynamoDB), the codebase can be used as starting point: https://github.com/0xflair/rockset-java-client
  - Official Flink has a DynamoDB connector BUT it only supports writing not reading from dynamodb, but can be used to get inspiration: https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/connectors/table/dynamodb/ and source code: https://github.com/apache/flink-connector-aws/tree/v4.1/flink-connector-dynamodb
* This library MUST use dynamodb "Query" to fetch the data (NOT "Scan") and must use the proper index that allows querying by namespace and entityType). Read the table structure for actul column names.
* The connector must work with Apache Flink 1.17

#### How to test?

1. To have a test DynamoDB you can either create an AWS account use their free DynamoDB cloud, OR bring up a local instance of dynamodb (https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) 
2. You can bring up a local Flink environment, and create a basic "batch" SQL job that queries a test data source and for example writes the result to a local CSV file. You can try their tutorial here: https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/try-flink/local_installation/
3. Now you can develop and bulid your connector, add it to Flink and then change your flink SQL test to read from "your connector" instead of a test data source, and still write to a CSV file.
4. When 1 and 2 are done you can provide the JAR file of the connector to our team and we will test it with our real DynamoDB table and within our real infrastructure and give you feedback.

### Table

* The table has this structure:
  - `etpid` (entityType + # + entityId) **Primary key / Partition key**
  - `ns` (namespace) **Sort key**
  - `nsetphr` (namespace + entityType + horizon)
  - ... more fields depending on customers custom entities ...
* The table has these indexes (which can be changed if needed):
  - `ns_nsetphr_index`

#### Virtual fields
There are certain virtual fields that exists on all rows:
* "namespace" is basically customer's name (e.g. sushiswap) stored in `ns` and `nsetphr`
* "entityType" is the type/model of this entity (e.g. Swap, or Transfer, or User) stored in `etp` and `nsetphr` and `entityType`
* "entityId" unique ID of this entity. Uniqueness is guaranteed on namespace+entityType level (i.e. two types can have entities with same id) stored in `etpid` and `entityId`
* "horizon" is a chronologically ordered string that may or may not be unique (e.g. 00000000000215897013-0000000005-0000000003-0000000016-00008), it is stored in `nsetphr` and `horizon` it consists of these parts:
  * `blockNumber` (e.g. 00000000000215897013) this is block number of the entity which can be used for partitions.
  * `forkIndex` (e.g. 0000000005) it can be from 0000000000 to 9999999999
  * `transactionIndex` (e.g. 0000000003) it can be from 0000000000 to 9999999999
  * `logIndex` (e.g. 0000000016) it can be from 0000000000 to 9999999999
  * `localIndex` (e.g. 00008) it can be from 00000 to 99999

## More questions?

This doc is very limited, we can discuss more details in a call :)
