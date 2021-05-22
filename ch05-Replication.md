# Chapter5 Replication

Replication means keeping a copy of the same data on multiple machines that are connected via a network.

Why we need to replicate data:
  * Reduce latency: To keep data geographically close to users.
  * Fault tolerance, Increased availabity: Allow system to continue working even if some of its parts have failed.
  * Increase throughput: Scale out num of machines that can serve read queries.

If data is not changing once written, replication is very simple, it just need to write the same data on to other nodes and its done. However, if data changes, managing replication can be complicated.

3 popular algorithms for replicating changes between nodes:
  1. Single-Leader
  2. Multi-Leader and 
  3. Leaderless

 Many trade-offs to consider with replication:
   * whether to use synchronous or asynchronous replication?
   * how to handled failed replicas?

## Leaders and Followers
With multiple replicas, how do we ensure all the written data ends up in all the replicas? 
Every write to the database, needs to be processed by every replca; otherwise replicas would not contain same data.
One common solution for this problem is LEADER BASED REPLICATION (a.k.a active/passive master/slave replication).

Leader based replication works as below:
  1. One of the replicas is designated as LEADER (master or primary). Client will send request to the leader, which gets written to leader's local storage first.
  2. Other replicas are called FOLLOWERS (slaves, secondaries or hot-standby). After leader write onto its storage, it sends out the change log on to all of its followers. Each follower takes this log from leader abd update its local copy.
  3. Client reads either from the leader or any of the followers. Writes are accepted only on the leader. Followers are read-only from client's point of view.

 ## Synchronous vs Async Replication
 In some relation DBs, this is often configurable option, other systems hardcoded to one of them.
   * Synchronous: Leader waits until follower has confirmed that it received the write before reporting success to the client and before making the write visible to subsequent reads.
     * Advantage of Synchronous replication is that the follower is guaranteed to have an up-to-date copy which is consistent with the leader.
     * Disadvantage - If follower doesnt respond, the write cannot be processed. Leader must block all writes and wait until failed replica is available again.
   * Asynchronous: Leader sends message, but does not wait for a response from follower.

It's impactical for all the followers to be synchronous: any one node outage would cause the whole system to grind to a halt. 

In practice, enabling synchrounous replication on a db, it usually means one of the followers is sync and others are async. If sync follower becomes unavailable or slow, one of the async followers is made sync. This guarantees that you have an up-to-date consistent copy of the data on atleast 2 nodes - the leader and one sync follower. This is called SEMI-SYNCHRONOUS replication.

In case of completely async replication, if the leader fails and is not recoverable, any writes that have not yet been replicated to followers are lost. This means no durability. 

Weakening durability may sound like bad trade-off, but async replication is widely used, especially if there are many followers.

## Setting up new Followers
When new followers need to be added  (either for increasing replicas or to replace failed nodes), how to ensure the new follower has accurate copy of leader's data?
Simply copying data files from one node to other is not sufficient: clients are constantly writing to DB and the data is always in flux.
Ideal solution is to do following:
  1. Take a consistent snapshot of the leader's DB at some point in time. (Simillar to taking a backup)
  2. Copy the snapshot on to new follower node
  3. Follower connects to leader and requests all the data changes that have happened since the snapshot was taken. This requires the snapshot is associated with a position - log sequence num (in MySQL its BINLOG COORDINATE ).
  4. When the folloer has processed the backlog of data changes since snapshot, we say it has caught up. It can now continue to process data changes from leader as they happen.

## Handling Node Outages

How to achieve HIGH AVAILABILITY with leader-based replication?

### Follower failure (CATCH-UP RECOVERY)
Each follower keeps log of data changes it has received from leader. If a follower crashes and is restsrted, or if network connection is temporarily interrupted, the follower can recover from its logs - it knows the last transaction that was processed before the fault. Thus follower can connect to the leader and request all the data changes since the failure.

### Leader failure (FAIOVER)
Handling failure of the leader is tricky: 
  1. One of the follower needs to be promoted as the new leader.
  2. Clients need to be reconfigured to send writes to this new leader and 
  3. Other followers need to start consuming data changes from the new leader. 
This process is called FAILOVER.

Automatic failover:
  1. Determining that the leader has failed. 
    Heart beats with timeouts can be used to detect if the leader is failed due to crash/power outage/network issue etc.
  2. Choosing a new leader. 
    Election process.
  3. Reconfiguring the system to use new leader. 
    Clients now need to talk to new leader. If the old leader comes back, it might still beleive that it is a leader, not realizing that the other replicas have forced it to step down. System need to ensure the old leader beacomes a follower and recognize the new leader.

Since automatic failover is complex, some operations teams prefer to perform failovers manually!

## Replication Logs

### Statement based replication
The leader logs every write request that it executed and send that stetement log to its followers.
For relational db's this means that every INSERT, UPDATE or DELETE statement is forwarded to followers and each follower parses and executes that SQL statement as if it had been received from client.
This can break :
   * Any statement with non-determinitic  function such as NOW() to get the current datate and time or RAND() to get a random number likely generate a diff value on each replica.
   * Statements using auto-incrementing column, or if the statements depending on the existing data (eg: UPDATE ... WHERE [some condition]), they must be executed in the same order on each replica or else they may have a different effect.

Older version of MYSQL used this but discouraged as this has above issues.

### Write-ahead log (WAL)
Low level logs capturing each change to DB.
* WAL is an append-only sequence of bytes containing all writes to the DB.
* The same log can be used to build a replica on another node
* One prblem with this problem - WAL contains details of which bytes were changed in which disk blocks. This makes the replication closely coupled to the storage engine. If DB changes its storage format from one version to another.

### Logical (row-based) log replication
To decouple from the storage engine formats and internals, this format of log uses - LOGICAL LOG. A LOGICAL LOG for a DB is usually a sequence of records describing writes to DB tables at a granularity of a row:

MySQL's BINLOG (when configured t use row-based replication) uses this approach.

### Trigger based replication
TBD

## Problems with Replication Lag

For workloads with mostly reads and only small writes (a common pattern on web) - create many followers, distribute read requests across the followers. This removes load from leader and allows read requests to be served by nearby replicas.

Problems with SYNC replication:
* In this READ-SCALING architecture, we can increase capcity for serving read-only requests simply by adding more followers.
* This approach only realistically works with asynchronous replication.
* If we try synchronous replication to all followers, a simgle node failure or network outage would make the entire system unavailable for writing.
* More nodes we have, the likelier it is that one will be down, so a fully sunchronous config would be very unreliable.

Problems with ASYNC replication:
* With asynchronous follower, it may see outdated information if follower has fallen behind.
* Inconsistencies - if we run the same query on the leader and a follower, may get different results, because not all writes have been reflected.
* This inconsistency is temporary. If we wait for a while, the followers will eventually catch up and become consistent with leader - EVENTUAL CONSISTENCY.

### Reading Your Own Writes
Lags in replication, can cause stale data served. We need READ_AFTER_WRITE  CONSISTENCY, also known as READ_YOUR_WRITES CONSISTENCY. This is a guarantee that if the user relaods the page, they will always see updates they submitted.
How to implement:
* Read it from leader - always read user's own profile from leader.
* Limit such reads from leader to may be first one minute to compensate replication lag.

### Monotonic Reads
It may happen that user see things moving backward in time due to replication lag (first read from a follower which had the record replicated, next read served from a replica which was yet to be replicated). MONOTONIC READS is to guarantee such anomaly does not happen.

One way to achieve MONOTONIC READ is to kake sureeach user always makes reads from the same replica  - for ex, the replica can be chosen based on the hash of UserID.

### Consistent Prefix Reads
Violation of causality. CONSISTENT PREFIX READS guarantees if a sequence of writes happens in a certain orfer, then anyone reading those writes will see them in same order.

Solution to this problem is to make sure writes that are causally related to each other are writtem to the same partition.


## MULTI-LEADER REPLICATION
Downside of Leader based apps: Only one leader, all writes must go through it. If leader is not reachable, we can;t write to DB.
Extension of leader-based model is to allow more than one node to accept writes. Replication can still happen the same way. This is called MULTI-LEADER config (master-mater active/active replication).

Use cases for multi leader:
* Multi-datacenter operation - leader in each data center geographically separated.
* Clients with offline operation - calendar app on mobile phone/laptop. Syncs with server when combes back online.
* Collborative editing - Google docs.

## Handling Write Conflicts
* Biggest problem with MULTI-LEADER replication is that WRITE CONFLICTS can occur. Conflict resolution is required.
* This problem does not occur in single-leader db. 

In Single-leader setup, second writer will either block or wait for the first write to complete or abort. on milt-leader both writes are successful and the conflict is only detected asynchronously at some later point of time.

### Conflict avoidance
Simplest strategy to deal with conflicts is to avoid them - All writes for a particular record go through same leader, then conflicts cannot occur.

### Converging towards a consistent state
In a multi-leader config, there is no defined ordering of writes, so its not clear what the final value shoulf be.
Few ways of achiveing convergence:
* LWW - LAST WRITE WINS - Give each write a unique ID (ex: timestamp or a UUID). Pick the write with highest UD as the winner. Even though its popular, it's prone to DATA LOSS.
* Each replica will a unique ID and let writes that originated at higher numbered replica always take precedence. Also implies DATA LOSS.
* Somehow merge the values together. Order alphabetically and concat.
* Record the conflict in a data structure, that preserves the info and provide option to resolve conflict programmatically.

### Automatic Conflict Resolution
* CRDTs - CONFLICT-FREE REPLICATES DATATYPES
* MERGEABLE PERSISTENT DATA STRUCTURES


## LEADERLESS REPLICATION
Abandons the concept of a LEADER, allowing any replica to directly accept writes from clients. Dynamo, Riak, Cassandra are some of LEADERLESS models. A coordinator is involed or the clients will have the resposibility of writing to all replicas.

* No failover in leadrless config as each replica acts as a leader.
* A read request is sent to several nodes in para,,e,.
* Client may get diff response from diff nodes. Version nums are used to determine which is newer.

#### Read repair
Eventually all data is copies to every replicas.
When a client reads a replica and figures its stale, it can issue a write to update it.

#### Anti-Entropy process
Run background process to constantsly look for differences in data b/w replicas and copies any missing data.

### Quorums
* `n` replicas 
* every write must confirmed by `w` nodes to be considered successful.
* Must query `r` nodes for read.
* As long as `w + r > n` we expect to get uptodate value when rerading.
This is because atleast one of `r` nodes we're reading from must be up to date.

Common choice:
* `n` an odd number (ex: 3 or 5)
* set `w = r = (n+1)/2`

The Quorum condition `w+r > n` allows the system to tolerate unavailable nodes as below:
* If `w < n`, we can still process if a node is unavailable.
* If `r < n`, we can still process reads if a node is unavailable.
* If n=3, w=2, r=2, we can tolerate one unavailable node.
* If n=5, w=3, r=3, we can tolerate two unavailable nodes.

### Sloppy Quorums and Hinted Handoff

SLOPPY QUORUM: A network interruption can easily cut off a client from a large number of nodes. Clients cannot connect to them. In this situation, its likely that fewer `w` or `r` reachable nodes remain, so the client can no longer reach QUORUM. Instead of failing writes, we accept writes and write them to some nodes that are reachable but are not among `n` nodes.

HINTED HANDOFF: Once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to appropriate `home` nodes.

SLOPPY QUORUM are practicularly useful for increasing WRITE AVAILABILITY.







