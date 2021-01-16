## General words about the system
ClickHouse is a columnar database management system (DBMS) for online analytical query processing (OLAP). Which is developed by Yandex and the open-source community.

In this chapter, we will deal with the specification and verification of the replication protocol details of this system.

### Distributed
The ClickHouse data model operates with tables. Each table is implemented by a specific engine, which is responsible for the data storage mechanism and the process of processing client requests.

ClickHouse supports horizontal scaling using distributed sharded tables, which are implemented by the *Distributed* engine.

ClickHouse is a fault-tolerant system where each shard of a distributed table is independently replicated, the replication protocol is encapsulated in the *Replicated* engine family.

### Replication
We will consider a separate shard of a distributed table.
* A shard is a set of replicas whose behavior is determined by the ReplicatedMergeTree engine
* Insertions into the shard table are done in blocks
* Inserts are performed through different shard replicas, replicas learn about the insertion to other nodes through ZooKeeper and download the corresponding block directly from other replicas that are currently connected to ZK
* To detect "dead" replicas, ZooKeeper stores an ethereal node is_active, which indicates whether a node has a connection with ZooKeeper. Let's call such replicas *active*.
* Blocks with data are stored in separate sorted files; these files need to be merged to optimize reading.
* In order for replicas to converge to one state, they must agree on the order of insertions and merges, for this they use the update log in ZooKeeper
* Information about the insert goes to the log after the replica has processed the user's request and written the data to its disk.
* Merges are assigned (added to the log) by the leader replica, which is selected using ZK.

### Clearing the log
It is impossible to store the entire update log in ZooKeeper indefinitely, you just need to maintain an acute tail, and old records that have already been processed by all replicas can be deleted.

We can delete the old commands that processed all the replicas, since the existing replicas will never see them, because they move their iterator only forward, and the new ones will copy the state from active replicas at startup.

Each of the replicas stores a log_pointer in the ZooKeeper, which points to the last record processed by the replica. During cleaning, we will delete entries from the log to the minimum log pointer of active replicas.

The replica may reconnect with the ZK and become active again. At this point it needs to understand if the entries it needs have been removed from the log. To do this, during log cleanup we will mark inactive replicas, for which we have deleted desired entries, with the is_lost flag in ZooKeeper.

When such a replica becomes active again, it can figure out whether it needs to copy the state from another replica or if it can just keep running the log.

### Quorum Insert / Select
In normal mode, the replica acknowledges the insertion to the client after it has written data to the local disk only, replication is done asynchronously.

The client wants to have more reliability receive an insert confirmation only after a synchronous write to multiple replicas. To do this, ClickHouse uses the *quorum inserts* mode.

The replica on which the quorum insert arrived creates a node in ZooKeeper, writes itself to it, and then adds a record of the quorum insert to the log. When other replicas reach this record, they will check to see if the insert has reached quorum. If not, they will download the corresponding block for themselves and join the quorum by updating the corresponding node in ZK.

If the replica sees that the correct number of replicas have gathered in the znode for the quorum, the node will delete that znode and the entry will be confirmed to the client. If for some reason the replica that wants to participate in the quorum cannot get that block, then the quorum is considered to have failed and information about this is written to ZooKeeper.

This algorithm can be improved to provide sequential consistency for reads. To achieve this, it is necessary to respond during reading only if the replica has all the blocks inserted with quorum. To do this, ZooKeeper stores the number of the last recorded block along with the quorum.

When Select comes to the replica, the replica responds to it only if it has all the blocks that were inserted with quorum, except for those for which the quorum has failed.

## Modeling
First of all, let's choose an abstraction level for the model.

During an update (insert or merge), the first thing a replica does is insert information about it into ZooKeeper. Thus, all actions on the system are ordered by the log and the information about new updates and the order on them is obtained from it, so, unlike Kafka, we do not need to model communication between replicas, and can assume that if a replica reads the information about the insertion, it immediately downloaded this piece of data to itself.

The main actors will be replicas and cliens. For better readability, let's combine the actions for each of the entities:

```
ReplicaAction ==
    \E replica \in Replicas:
        \/ ExecuteInsert(replica)
        \/ ExecuteMerge(replica)
        \/ BecomeLeader(replica)

ClientAction ==
    \/ Insert
    \/ Select

```

Replicas can restart or lose connectivity with other replicas and ZK (for example, if the replica is in a DC that is inaccessible due to channel damage). We will simulate this using the ReplicaBecomeInactive and ReplicaBecomeActive actions:

```
ReplicaBecomeInactive ==
    /\ \E replica \in Replicas :
      /\ IsActive(replica)
      /\ RepicaStartInactive(replica)
      ...

ReplicaBecomeActive ==
    /\ \E replica \in Replicas :
      /\ ~IsActive(replica)
      /\ RepicaStartActive(replica)
      ...
```

All information about the replica is stored (is_active flag, log_pointer, local chunks, etc.) in a separate node in ZK. After restarting, the replica must restore which log entries it has already processed and which pieces it has locally.

Non-determinism in the system manifests itself in several points:

The client can send its request to any of the replicas. We simulate sending requests using an action:

```
QuorumReadLin ==
    ...
    /\ \E replica \in Replicas :
        /\ IsActive(replica)
        /\ Cardinality(GetCommitedId) = Cardinality(replicaState[replica].local_parts \ ({quorum.data} \cup GetData(failedParts)))
    ...

ClientAction ==
    \/ QuorumInsert
    \/ QuorumReadLin
```

Secondly, any of the replicas can take an action on its own initiative: refuse, process a new entry in the log, update the quorum information. We simulate the replica behavior using the ReplicaAction:

```
 ReplicaAction ==
     \E replica \in Replicas:
      \/ IsActive(replica)
          /\ \/ ExecuteLog(replica)
             \/ UpdateQuorum(replica)
             \/ EndQuorum(replica)
             \/ BecomeInactive(replica)
             \/ FailedQuorum(replica)
      \/ ~IsActive(replica)
          /\ BecomeActive(replica)
```

### Testing
Verification confirms the correctness of the log trimming algorithm: the ValidLogPointer invariant is not violated:

```
ValidLogPointer == [] (\A replica \in Replicas: IsActive(replica) => deletedPrefix < replicaState[replica].log_pointer)
```

A spec about Quorums deserves special attention.

There is a sequential_consistency setting in ClickHouse, which should provide one of the consistency modes.

In this mode, the replica responds to a user request when its maximum local last inserted block number is equal to the number of the last block inserted using quorum inserts, which is read from ZK. Reads, in ZK, unlike writes, do not go through the ZAB protocol, but are performed locally. This means that we could get an obsolete value for the last quorum entry.

```
QuorumReadWithoutLin ==
    /\ Len(log) > 0
    /\ Cardinality(GetSelectFromHistory(history)) < HistoryLength
    /\ \E replica \in Replicas :
        /\ IsActive(replica)
        /\ LET max_data == Max(replicaState[replica].local_parts \ ({quorum.data} \cup GetData(failedParts)))
            IN /\ max_data # NONE
              /\ ReadEvent(GetIdForData(max_data, log))
    /\ UNCHANGED <<log, replicaState, nextRecordData, quorum, lastAddeddata, failedParts>>
```

If you check the history, then TLC will give an error.

If ZK makes the reads linearizable, then ClickHouse itself will also be linearizable for quorum readings. Let's simulate this with the QuorumReadLin action:

```
QuorumReadLin ==
    /\ Len(log) > 0
    /\ Cardinality(GetSelectFromHistory(history)) < HistoryLength
    /\ \E replica \in Replicas :
        /\ IsActive(replica)
        /\ Cardinality(GetCommitedId) = Cardinality(replicaState[replica].local_parts \ ({quorum.data} \cup GetData(failedParts)))
        /\ LET max_data == Max(replicaState[replica].local_parts \ ({quorum.data} \cup GetData(failedParts)))
           IN /\ max_data # NONE
              /\ ReadEvent(GetIdForData(max_data, log))
    /\ UNCHANGED <<log, replicaState, nextRecordData, quorum, lastAddeddata, failedParts>>
```

If you check the history with such an action, it turns out to be linearizable.
