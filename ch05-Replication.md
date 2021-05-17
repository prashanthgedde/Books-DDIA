# Chapter5 Replication

Replication means keeping a copy of the same data on multiple machines that are connected via a network.

Why we need to replicate data:
  * Reduce latency: To keep data geographically close to users.
  * Fault tolerance, Increased availabity: Allow system to continue working even if some of its parts have failed.
  * Increase throughput: Scale out num of machines that can serve read queries.

If data is not changing once written, replication is very simple, it just need to write the same data on to other nodes and its done. However, if data changes, managing replication is bit complex.
3 popular algorithms for replicatig changes between nodes:
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
  1. One of the replicas is designated as LEADER (master or primary). A write will send requests to the leader, which gets written to leaders local storage first.
  2. Other replicas are called FOLLOWERS (slaves, secondaries or hot-standby). After leader write onto its storage, it sends out the change log on to all of its followers. Each follower takes this log from leader abd update its local copy.
  3. Client reads either from the leader or any of the followers. Writes are accepted only on the leaser. Followers are read-only from client's point of view.

 ## Synchronous vs Async Replication
 In some relation DBs, this is often configurable option, other systems hardcoded to one of them.
   * Synchronous: Leader waits until follower has confirmed that it received the write before reporting success to the client and before making the write visible to subsequent reads.
     * Advantage of Synchronous replication is that the follower is guaranteed to have an up-to-date copy which is consistent with the leader.
     * Disadvantage - If follower doesnt respond, the write cannot be processed. Leader must block all writes and wait until failed replica is available again.
   * Asynchronous: Leader sends message, but does not wait for a response from follower.

It's impactical for all the followers to be synchronous: any onr node outage would cause the whole system to grind to a halt. 

In practice, enabling synchrounous replication on a db, it usually means one of the followers is sync and others are async. If sync follower becomes unavailable or slow, one of the async followers is made sync. This guarantees that you have an up-to-date consistent copy of the data on atleast 2 nodes - the leader and one sync follower. This is called SEMI-SYNCHRONOUS replication.

In case of completely async replication, if the leader fails and is not recoverable, any writes that have not yet been replicated to followers are lost. This means no durability. 

Weakening durability may sound like bad trade-off, but async replivation is widely used, especially if there are many followers.

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
Handling failure of the leader is tricker: 
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

### Logical (row-based) log replication

### Trigger based replication


