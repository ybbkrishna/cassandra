How data is stored in cassandra ?

- All the data is associated with a token which is a 64 bit integer range 2^-63 - 2^63
- When only one node, then it will be responsible for all the token values
- When a new node is added it will take on responsibility of contiguous set of token values, this process continues for more nodes.
- Cassandra 1.2 introduced a concept of vnodes.

- Virtual nodes (vnodes)
    - Each node can be represented by multiple vnodes of smaller size
    - benefits
        - Each node doesn't need to be take responsibility of a contiguous set of tokens
        - Each vnode take responsibility of contiguous set of tokens
    - Each node can have configurable number of virtual nodes, So a node deployed on more powerful hardware can have more vnodes and a node with less powerful hardware can have less vnodes
- Cassandra is a partition row store which means all the data is read or wrote with a partition key
- Cassandra.yaml contains the all the config options and it is available in the path: /data/conf/cassandra.yaml
    - endpoint_snitch - this option defines the type of snitch to use
- /data/conf/cassandra-rackdc.properties contains datacenter and rack related info.

Commands

- nodetool (helps in administration)

Bhargav’s MacBook Pro ~: docker exec -it n1 nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens  Owns (effective)  Host ID                               Rack
UN  172.17.0.2  51.49 KB   256     100.0%            cd526aba-0ea2-4bed-8621-f6968794a393  rack1

Columns

- #1 - UN - Up and Normal
- #4 - Tokens - Number of virtual nodes
- #5 - Owns - %ge of token values in the range ( 2^-63 - 2^63) it holds

Bhargav’s MacBook Pro ~: docker exec -it n1 nodetool ring

Datacenter: datacenter1
==========
Address     Rack        Status State   Load            Owns                Token
                                                                           9022365958691174818
172.17.0.2  rack1       Up     Normal  51.49 KB        100.00%             -9194860533977059948

Columns

- #1 - Address - address of the node it resides in
- #7 - Token - represents the end of the token value it is responsible for

docker run --name=n3 -d tobert/cassandra -dc DC2 -rack RAC1 -seeds 172.17.0.2

Bhargav’s MacBook Pro ~: docker exec -it n1 nodetool describering uber
TokenRange(start_token:1310874187927888253, end_token:1325357923752612385, endpoints:[172.17.0.4, 172.17.0.3, 172.17.0.2], rpc_endpoints:[172.17.0.4, 172.17.0.3, 172.17.0.2], endpoint_details:[EndpointDetails(host:172.17.0.4, datacenter:datacenter1, rack:rack1), EndpointDetails(host:172.17.0.3, datacenter:datacenter1, rack:rack1), EndpointDetails(host:172.17.0.2, datacenter:datacenter1, rack:rack1)])

nodetool repair

- this command helps in repairing the failed data.

nodetool pausehandoff

- This pauses the hinted handoff

Snitch

- This is used by cassandra to gain and understanding of the environment
- Used to efficiently route the requests.
- Consulted when writing multiple copies of the data
- Simplesnitch
    - used in single datacenter
- Gossiping Propertyfile Snitch
    - Uses a property file deployed with each node containing the node's location, then the nodes gossip this information to each other
    - This can differentiate two nodes in different racks or different data centres
- it teaches Cassandra enough about your network topology to route requests efficiently
- it allows Cassandra to spread replicas around your cluster to avoid correlated failures. It does this by grouping machines into "datacenters" and "racks."  Cassandra will do its best not to have more than one replica on the same "rack" (which may not actually be a physical location)

Cassandra Terminology

- Keyspace
    - analogous to Oracle/mySql table space
- One or more tables
- Partitions
    - all data written to Cassandra is associated with a partition key. which plays a major role in where the data is stored.
    - Primary interaction point to read/write data from/to Cassandra
- Row

Replication and Consistency

- Replication strategy is determined at the keyspace level.
- Partition key is used to determine the location of 1st copy of the data, where as  replication strategy at keyspace is used to determine the number of replicas as well as their  location.

Replication Strategies

- SimpleStrategy
    - cql - `create keyspace uber with replication = {‘class’: ’simpleStrategy’, ‘replication_factor’: 3};`
    - This can be used during development or single datacenter cluster
- NetworkToplogyStrategy
    - cql - `create keyspace uber with replication = {‘class’: ’networkTopologyStrategy’, ‘DC1’: 3, ‘DC2’: 1};`

Reads and Writes

- Coordinator node
    - a client when connected to a cluster will be connected to a node that node is called a coordinator node.
- Tunable write consistency
    - This is a setting which requires n number of replica writes to be successful for the write to be successful from the coordinator to the client
    - This can be set at keyspace level or even at the table level
    - If set to ONE then coordinator node returns success once a replica responds with a success ack.
    - if set to QUORUM then coordinator node returns success once majority of replicas responds with success ack.
    - if set to ALL then coordinator node returns success once all the replicas respond with success ack.
        - If consistency is set to all and if one of the replica is down then both read and write will fail.
    - If set to ANY then coordinator node returns success if write can make it to the cluster even to the coordinator node.
- Hinted Handoff
    - If node is unavailable during write then coordinator node saves the message and retries till it gets succeeded.
    - If both coordinator and replica node are down, then no one knows about the update that needs to be written to the replica node.
- Tunable read consistency
    - It is the number of nodes coordinator consults before returning the result.
    - Coordinator gets data from one node and digest from other replica nodes (number depends on the consistency level)
- Strong Consistency
    - write consistency + read consistency > replication factor
- Multidatacenter consistency
    - if set to EACH_QUORUM then coordinator returns success iff majority of nodes in each datacenter gives a success ack.
    - if set to LOCAL_QUORUM then coordinator returns success if quorum is met in the DC where coordinator is located.v- preferred
    - if set to LOCAL_ONE is same as level ONE but the data is read from the DC where the coordinator is located.

 CQL
https://docs.datastax.com/en/cql/3.1/cql/cql_reference/cqlReferenceTOC.html
Check Cql help and cql help topics

cqlsh commands
https://docs.datastax.com/en/cql/3.1/cql/cql_reference/cqlCommandsTOC.html
https://docs.datastax.com/en/cql/3.1/cql/cql_reference/cqlshCommandsTOC.html

- create keyspace

create keyspace uber with replication = {‘class’: ’simpleStrategy’, ‘replication_factor’: 3};

- use keyspace;

use uber

- create table

create table trips (id varchar primary key);

- get current consistency level

consistency;

- set consistency level

consistency quorum;

- insert data into table

insert into trips (id) values (‘trip-1’);

- enable logs, debug logs or verbose

tracing on

- alter keyspace

alter keyspace uber with replication = {‘class’: ’NetworkTopologyStrategy’, ‘DC1’: 3} AND DURABLE_WRITES  = true;

- removing or dropping a keyspace

drop keyspace uber;

- alter table

alter table uber.trips ADD user varchar;
alter table uber.trips DROP title;

- remove all the data from the table

truncate uber.trips;

- drop or delete table

drop table trips

Table Properties
 https://docs.datastax.com/en/cql/3.1/cql/cql_reference/tabProp.html

- Setting DURABLE_WRITES to false during keyspace creation skips writing to the commit log while writing to the tables.

Basic Data types

- https://docs.datastax.com/en/cql/3.1/cql/cql_reference/cql_data_types_c.html

Naming restrictions for keyspaces, tables and columns

- No hyphens
- No spaces
- Double quotes required if it starts with a number “2015stats"
- Double quotes required for the mixed quoted names else they will be flattened to lowercase

Commands during demo

cqlsh> create keyspace with replication = {'class': 'SimpleStrategy', 'replication_factor':1};
SyntaxException: <ErrorMessage code=2000 [Syntax error in CQL query] message="line 1:16 no viable alternative at input 'with' (create keyspace [with]...)">
cqlsh> create keyspace uber with replication = {'class': 'SimpleStrategy', ‘replication_factor'}
cqlsh> use uber;
cqlsh:uber> create table trips (id varchar primary key);
cqlsh:uber> create table trips (id varchar primary key);
AlreadyExists: Table 'uber.trips' already exists
cqlsh:uber> create table if not exists trips (id varchar primary key);
cqlsh:uber> alter table trips add duration int;
cqlsh:uber> alter table trips add started_at timestamp;
cqlsh:uber> alter table trips add user varchar;
cqlsh:uber> alter table trips with comment = 'A table for trips’;
cqlsh:uber> desc table trips

CREATE TABLE uber.trips (
    id text PRIMARY KEY,
    duration int,
    started_at timestamp,
    user text
) WITH bloom_filter_fp_chance = 0.01
    AND caching = '{"keys":"ALL", "rows_per_partition":"NONE"}'
    AND comment = 'A table for trips'
    AND compaction = {'min_threshold': '4', 'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy', 'max_threshold': '32'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND dclocal_read_repair_chance = 0.1
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';

cqlsh:uber> drop table trips
        ... ;
cqlsh:uber> create table trips (
        ... id varchar primary key,
        ... name varchar,
        ... name varchar,
cqlsh:uber> create table courses (
        ... id varchar primary key,
        ... name varchar,
        ... author varchar,
        ... audience int,
        ... duration int,
        ... cc boolean,
        ... released timestamp
        ... ) with comment = 'A table from courses at uber';
cqlsh:uber> desc table courses

CREATE TABLE uber.courses (
    id text PRIMARY KEY,
    audience int,
    author text,
    cc boolean,
    duration int,
    name text,
    released timestamp
) WITH bloom_filter_fp_chance = 0.01
    AND caching = '{"keys":"ALL", "rows_per_partition":"NONE"}'
    AND comment = 'A table from courses at uber'
    AND compaction = {'min_threshold': '4', 'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy', 'max_threshold': '32'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND dclocal_read_repair_chance = 0.1
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';

Selecting data

- Select query with out a where clause will hit all the nodes holding data from this table
    - this can be observed with tracing on.
- WHERE, ORBER BY clause can filter only the primary key columns and partition key columns are mandatory
- DISTINCT clause can be done only on partition key columns or static columns.

- its all upserts even if its insert/update
- deleting a column is similar to updating that column as null
- Expiring data with TTL
    - update uber.courses USING TTL 32400 SET reset_token = ’sdfsdf’ WHERE id = ’sdfa'
- Retrieving the set TTL
    - select ttl(reset_token) from uber.courses where id = ’sfda'
- writetime function in select statement gives the datatime of the column specified
    - select writetime(author) from uber.courses;
- function in select statement which gives the token key value
    - select token(id) from uber.courses;
- Setting TTL for a row
    - insert into uber.reset_token (id, token) VALUES (‘john’, ‘asdfads’) using TTL 10800;
- setting Table wide TTL
    - create table reset_tokens (id varchar primary key, token varchar) with default_time_to_live = 10800
- Deleting data in cassandra
    - when data is deleted with consistency level as quorum a tombstone is created in all the places where the data resides
    - If some node is lagging the delete update, hintedhandoff (read repair) will take care of propagation of these changes once the node is up.
    - These tombstones are periodically purged from the db
    - Purging shouldn't be done frequently as it can lose the changes and cannot make them once the failed node is up.
    - gc_grace_seconds determine the purge interval of the tombstones. can be specified per table basis.

Counters

- Counters must reside inside tables other than primary keys.
- They carry some overhead as they follow read before write semantics.

MultiRow Partitions
Defining primary key can be done in many different ways

//single column
CREATE TABLE courses (
    id varchar PRIMARY KEY
)

CREATE TABLE courses (
    id varchar,
    PRIMARY KEY (id)
)

- single column primary key
    - PRIMARY KEY (id)
- first item in the primary key is treated as a partition key
- primary key method definition
    - PRIMARY KEY(partition keys, clustering_keys)
        - (partition keys, clustering_keys) => composite key
- if defined like PRIMARY KEY(a,b,c,d)
    - a will be partition key and b,c,d are clustering_keys
- we can have multiple keys as partition keys
    - PRIMARY KEY((a,b,c),d,e,f)
        - in here a,b,c are partition keys
    - PRIMARY KEY ((a,b,c))
        - Now every unique combination of (a,b,c) is in different partition

Static Columns

- These can be used for the columns whose values are constant across all rows in the partition.
- The major use comes in reducing data duplication.
- This can be used when declaring parent’s variables.

Time series Data

- example sources
    - clickstream data
    - IOT data..
- Cassandra helps in storing the data via timeuuid data type
    - version 1 of the UUID contains
        - No of 100 ns intervals since UUID epoch
        - MAC address
        - CLOCK seq numbers to prevent duplicates
    - These are guaranteed unique and sortable by date and time

TimeUUID functions

- now

INSERT INTO course_page_views (course_id, view_id)
VALUES (‘intro-golang’, now())

- https://docs.datastax.com/en/cql/3.3/cql/cql_reference/timeuuid_functions_r.html
- Cassandra can only hold 2 billion cells for a partition that is (rows * columns)

Bucketing Timeseries data

- Cassandra can only hold 2 billion cells for a partition that is (rows * columns)
- Creating a table with partition key as month and year makes sure that we only store one month worth of data.

Complex Data types

- Other than primary data types cassandra supports
    - Collections
    - Tuples
    - User defined types

Collections

- types
    - Set
    - List
    - Map
- TTL on collections is applied to each element in the collection

Tuple - is a series of values that can contain any types but should be in the same order

- (varchar, int, int, varchar, varchar)

- Nesting complex datatypes can only be achieved using 'frozen'
    - last_login map<varchar, frozen<tuple<timestamp, inet>>>

User Defined Types

- All the user defined types currently require nesting
- Type will have the scope of keyspace

Making the Most of Cassandra

Secondary Indexes

- Cassandra supports the creation of secondary indexes

CREATE INDEX users_company ON users(company);

- Cassandra supports the secondary index creation on collections too
- Querying is a bit different when querying on collection secondary index

SELECT * FROM users WHERE  tags CONTAINS ‘java’;

- By default for maps cassandra matches values in the map
- To have keys mapped one should specify like

CREATE INDEX ON users (KEYS(last_login))
and while querying
SELECT * FROM users WHERE last_login CONTAINS KEY <key>

When not to use a Secondary Index

- High (or very low) cardinality columns
- Tables with counter columns - not supported
- Frequently updated or deleted columns
- Tables with very large partitions
    - Filtering to single partition then on secondary index should be fine though
- Static columns - not supported

Manually Maintained Indexes

- It will be a regular table modelled to support our usecase
- But requires a bit of effort from developer end

Batches

- Intended for keeping related tables in sync
- Not Intended for fast loading of data.
- It is a group of CQL statements, as it is processed by coordinator node, its progress is communicated to other nodes in the cluster. which helps the other node to take over the batch is coordinator node is down.
- This is not a transaction and so wont have any rollback related feature.

Unlogged Batches

- If all the data is going in to the same partition then we can use unlogged batch, then coordinator node will directly inform that partition, and will be a single write.
- Is appropriate when writing multiple rows to a single partition.

Alternatives to Batches

- For high speed data loads single loops on client should do.

Light Weight Transactions

- These are based on paxos algorithm
- Each transaction needs a leader and other as replica nodes
- It needs 4 round trip conversations between leader and replica nodes
    - Prepare <-> Promise
    - Read <-> Results
    - Propose <-> Accept
    - Commit <-> Ack
- 1, 3 steps are core to the paxos algorithm
- 2,4 are extensions to cassandra specific use case.
- Cassandra exposes these as functionality as COMPARE and SET operations.
    - These operations can help maintain concurrency control.
    - These can be applied on batches too but need to be on the first statement
    - If the first statement condition is not met then none of the other statements execute.
- Light weight transactions on batches can be used only if all the statements operate on same table.

Data Modeling

- Types
    - Formal
        - Conceptual data modelling - understanding the data
        - Query graph - understanding all the queries and access patterns
        - Logical data model - query driven model
        - Physical - apply optimisations and implement using cql
    - Practical
- Conceptual data modelling
    - data entities and relations
    - doesnt depend on the technology
    - graphical by er diagrams
