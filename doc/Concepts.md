# Concepts
We need to introduce some common terminologies that are used in Vitess:
### Keyspace
A keyspace is a logical database.
In its simplest form, it directly maps to a MySQL database name.
When you read data from a keyspace, it is as if you read from a MySQL database.
Vitess could fetch that data from a master or a replica depending
on the consistency requirements of the read.

When a database gets [sharded](http://en.wikipedia.org/wiki/Shard_(database_architecture)),
a keyspace maps to multiple MySQL databases,
and the necessary data is fetched from one of the shards.
Reading from a keyspace gives you the impression that the data is read from
a single MySQL database.

### Shard

A division within a Keyspace. All the instances inside a Shard have the same data (or should have the same data,
modulo some replication lag).

A Keyspace usually has one shard when not using any sharding (we name it '0' by convention). When sharded, a Keyspace will have N shards (usually, N is a power of 2) with non-overlapping data.

We support [dynamic resharding](Resharding.md), when one shard is split into 2 shards for instance. In this case, the data in the
source shard is duplicated into the 2 destination shards, but only during the transition. Afterwards, the source shard is
deleted.

A shard usually contains one MySQL master, and many MySQL slaves. The slaves are used to serve read-only traffic (with
eventual consistency guarantees), run data analysis tools that take a long time to run, or perform administrative tasks (backups, restore, diffs, ...)

### Tablet

A tablet is a single server that runs:
- a MySQL instance
- a vttablet instance
- a local row cache instance
- an other per-db process that is necessary for operational purposes

It can be idle (not assigned to any keyspace), or assigned to a keyspace/shard. If it becomes unhealthy, it is usually changed to scrap.

It has a type. The commonly used types are:
- master: for the mysql master, RW database.
- replica: for a mysql slave that serves read-only traffic, with guaranteed low replication latency.
- rdonly: for a mysql slave that serves read-only traffic for backend processing jobs (like map-reduce type jobs). It has no real guaranteed replication latency.
- spare: for a mysql slave not use at the moment (hot spare).
- experimental, schema, lag, backup, restore, checker, ... : various types for specific purposes.

Only master, replica and rdonly are advertised in the Serving Graph.

### Keyspace id
A keyspace id (keyspace_id) is a column that is used to identify a primary entity
of a keyspace, like user, video, order, etc.
In order to shard a database, all tables in a keyspace need to
contain a keyspace id column.
Vitess sharding ensures that all rows that have a common keyspace id are
always together.

It's recommended, but not necessary, that the keyspace id be the leading primary
key column of all tables in a keyspace.

If you do not intend to shard a database, you do not have to
designate a keyspace_id.
However, you'll be required to designate a keyspace_id
if you decide to shard a currently unsharded database.

A keyspace_id can be an unsigned number or a binary character column (unsigned bigint
or varbinary in mysql tables). Other data types are not allowed because of ambiguous
equality or inequality rules.

TODO: The keyspace id rules need to be solidified once VTGate features are finalized.

### Shard graph
The shard graph defines how a keyspace has been sharded. It's basically a per-keyspace
list of non-intersecting ranges that cover all possible values a keyspace id can cover.
In other words, any given keypsace id is guaranteed to map to one and only one
shard of the shard graph.

We are going with range based sharding.
The main advantage of this scheme is that the shard map is a simple in-memory lookup.
The downside of this scheme is that it creates hot-spots for sequentially increasing keys.
In such cases, we recommend that the application hash the keys so they
distribute more randomly.

For instance, an application may use an incrementing UserId as a primary key for user records,
and a hashed version of that UserId as a keyspace_id. All data related to one user will be on
the same shard, as all rows will share that keyspace_id.

### Replication graph
The [Replication Graph](ReplicationGraph.md) represents the relationships between the master
databases and their respective replicas.
This data is particularly useful during a master failover.
Once a new master has been designated, all existing replicas have to
repointed to the new master so that replication can resume.

### Serving graph
The [Serving Graph](ServingGraph.md) is derived from the shard and replication graph.
It represens the list of active servers that are available to serve
queries.
VTGate (or smart clients) query the serving graph to find out which servers
they are allowed to send queries to.

### Topology Server
The Topology Server is the backend service used to store the Topology data, and provide a locking service. The implementation we use in the tree is based on Zookeeper. Each Zookeeper process is run on a single server, but may share that server with other processes.

There is a global instance of that service. It contains data that doesn't change often, and references other local instances. It may be replicated locally in each Data Center as read-only copies. (a Zookeeper instance with two master instances per cell and one or two replicas per cell is a good configuration).

There is one local instance of that service per Cell (Data Center). The goal is to transparently support a Cell going down. When that happens, we assume the client traffic is drained out of that Cell, and the system can survive
using the remaining Cells. (a Zookeeper instance running on 3 or 5 hosts locally is a good configuration).

The data is partitioned as follows:
- Keyspaces: global instance
- Shards: global instance
- Tablets: local instances
- Serving Graph: local instances
- Replication Graph: the master alias is in the global instance, the master-slave map is in the local cells.

Clients usually just read the local Serving Graph, therefore they only need the local instance to be up. Also, we provide a caching layer for Zookeeper, to survive local Zookeeper failures and scale read-only access dramatically.

### Cell (Data Center)

A Cell is a group of servers and network infrastructure collocated in an area. It is usually a full Data Center, or a subset of a full Data Center.

A Cell has an associated Topology Server, hosted in that Cell. Most information about the tablets in a cell is hosted in that cell's Topology Server. That way a Cell can be taken down and rebuilt as a unit, for instance.

We try to limit cross-cell traffic (both for data and metadata), and gracefully handle cell-level failures (like a Cell being cut off the network). Having the ability to route client traffic to Cells individually is a great feature to have
(but not provided by the Vitess software).
