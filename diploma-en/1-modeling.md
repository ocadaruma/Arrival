# Part 1 (modeling)
### Reactivity
Unlike classical algorithms that receive an input and give a response in a finite time, distributed systems are reactive: they react in an infinite loop to requests from clients, signals from timers, and messages sent via an internal protocol.

To describe such a system means to describe its instantaneous state and a set of handlers for various events that lead to a change in state.

In TLA+, we describe the state of the system (all its constituent nodes and components) using a set of variables, and the transition between states (the system's response to events) - using actions.

For example, the state of the replicas in Raft includes:
* currentTerm - the number of the term in which the replicas are located
* state - the current role of the replica (leader / follower / candidate)
* votedFor - the vote cast during the leader selection phase in the current term

Examples of actions:
- BecomeLeader in Kafka - the controller has appointed the node as the new leader of the partition
- FollowerReplicate in Kafka - follower replicated a message from the partition leader's log
- Restart in Raft - the replica for some reason restarted and lost the data stored in RAM
- ClientRequest in Raft - the leader received a request from the user
- Commit in Snapshot Isolation - the transaction has been applied to the storage

The behavior of the entire system is described by the disjunction of handlers for all possible events. Let's take the Kafka spec as an example:
```
Next ==
    \/ ControllerElectLeader
    \/ ControllerShrinkIsr
    \/ BecomeLeader
    \/ LeaderExpandIsr
    \/ LeaderShrinkIsr
    \/ LeaderWrite
    \/ LeaderIncHighWatermark
    \/ BecomeFollowerTruncateToHighWatermark
    \/ FollowerReplicate
```

### Level of abstraction
A distributed system is individual nodes / microservices that play different roles, which interact by exchanging messages. But the external user usually does not observe any distribution, the nodes of the system are hidden from him behind a network address. At this address, the client sends commands and receives a response, usually with the help of a client library. For the user, the system is a concurrent object in the shared memory model, and the network address is essentially the name of that object.

Example - client interaction with the ZooKeeper system:

```
from kazoo.client import KazooClient

zk = KazooClient(hosts='127.0.0.1:2181')
zk.start()
zk.create("/my/favorite/node", b"a value")
```

The reasoning above can be applied not only to the system itself, but also to its individual components. For example: Kafka uses ZooKeeper to store quorum composition. For controller nodes, the quorum composition is one shared variable, but inside the ZK, this data is stored in several copies on different nodes and synchronized using the ZAB protocol.

The first step in developing a system's specifications is to select the level of detail for the components that make it up.

Let's compare the existing systems and their specs:

#### Paxos/Raft

In the specs of these algorithms, the highest level of detail is selected: the state of individual nodes of the distributed system and the protocol of their interaction are simulated, node restarts are simulated.

#### Kafka

In this system, the partition leader accepts new client records and writes them to the replica quorum (ISR, *in-sync replicas*). In the event of the death of the partition leader, the * controller * node appoints a new leader and a new quorum for writing via ZK, and sends notifications to replicas. ZK is needed for the global ordering of the leader / ISR changes.

When modeling the Kafka replication protocol in TLA+, ZK is not explicitly represented: ISR and leader are modeled as separate variables that the controller works with. For the controller, ZK is just a fault-tolerant shared memory, all the complexity of the system is hidden from it. Despite the fact that all quorum and leader updates are ordered in ZK, the replicas themselves receive notifications about these events over the network from the controller in an arbitrary order, so the spec explicitly simulates the interaction of replicas and network messages from the controller to the replicas.

#### Percolator
Percolator is a distributed client-side transaction protocol over BigTable -- distributed k/v storage. All interaction between clients takes place through BigTable, so the distribution is not explicitly modeled, transactions work in the shared memory model.

### Modeling participants
It is not always necessary to explicitly model entities that perform actions in the system.

Lamport gives an example:
> Mathematical manipulation of specifications can yield new insight. A producer/consumer system can be written as the conjunction of two formulas, representing the producer and consumer processes. Simple mathematics allows us to rewrite the same specification as the conjunction of n formulas, each representing a single buffer element. ... Processes are not fundamental components of a system, but abstractions that we impose on it.

For distributed systems, you can also model not the participants themselves, but the entities which they work with.

For example, in Paxos, the proposer runs an endless loop in which it picks a new ballot and tries to suggest a value. At the same time, there is no explicit cycle in the proposer spec; instead, separate proposals are modeled:

```
Next == \/ \E b \in Ballot : \/ Phase1a(b)
                             \/ \E v \in Value : Phase2a(b, v)
        \/ \E a \in Acceptor : Phase1b(a) \/ Phase2b(a)
```

Another example is the Spansphot Isolation spec, in which clients are not modeled, and transactions are born "out of thin air":

```
Next == \/ \E txn \in TxnId :
               (* Public actions *)
            \/ Begin(txn)
            \/ Commit(txn)
            \/ ChooseToAbort(txn)
            ...
```

### Modeling the world
Let's start modeling the world in which our system functions: we will describe how the network transmits messages, how time flows, how nodes fail.

In the theory of distributed computing, there are two main models: *synchronous* and *asynchronous*. In asynchronous, there are no restrictions on:
* message delivery time
* clock drift
* relative speed of nodes.

In a synchronous model, these parameters are limited.

Reality is somewhere in between. For example, a real network is synchronous most of the time, but no network infrastructure can guarantee this 100% of the time. There are periods of instability when the network delays and loses messages.

We will use the asynchronous model. We strongly pessimize reality; the real network is never asynchronous for an infinitely long time. On the other hand, if we make sure that the algorithm or system is correct in the most complex model, then in the real world we will get only correct behavior. Another reason for choosing the asynchronous model is that it is much easier to express it in TLA+.

Let's talk in more detail about each of the components of the model.

### Networking and messaging
The network through which the nodes of a distributed system interact are wires and intermediate devices: switches and routers. A message sent to the network exists either as a signal inside the wire, or as a set of bytes in the buffer of an intermediate device.

In TLA+, we can abstract away from these physical details and think of the network as simply a set of in-flight messages:

```
VARIABLE msgs
```

To send a message, add it to this set:

```
Send(m) == msgs' = msgs \cup {m}
```

Example:

The Proposer in Single-Decree Paxos sends a Prepare(b) message to acceptors:

```
Phase1a(b) == /\ Send([type |-> "1a", bal |-> b])
              /\ UNCHANGED <<maxBal, maxVBal, maxVal>>
```

The Acceptor receives it and sends back a Promise:

```
Phase1b(a) == /\ \E m \in msgs :
                  /\ m.type = "1a"
                  /\ m.bal > maxBal[a]
                  /\ maxBal' = [maxBal EXCEPT ![a] = m.bal]
                  /\ Send([type |-> "1b", acc |-> a, bal |-> m.bal,
                            mbal |-> maxVBal[a], mval |-> maxVal[a]])
              /\ UNCHANGED <<maxVBal, maxVal>>
```

In Kafka, the controller changes assigns a new leader and a new quorum, and then sends notifications to replicas of the partition:

```
ControllerUpdateIsr(newLeader, newIsr) == \E newLeaderEpoch \in LeaderEpochSeq!IdSet :
    /\ LeaderEpochSeq!NextId(newLeaderEpoch)
    /\  LET newControllerState == [
            leader |-> newLeader,
            leaderEpoch |-> newLeaderEpoch,
            isr |-> newIsr]
        IN  /\ quorumState' = newControllerState
            /\ leaderAndIsrRequests' = leaderAndIsrRequests \union {newControllerState}
```

### Network failures
An asynchronous network can delay messages for an arbitrary time, lose them or duplicate them, or even break up into unconnected segments (*partitions*).

There is no need to explicitly model delays or message losses: the model checker examines all possible trajectories, including those in which messages are not delivered for a long time or are not selected at all from the set of msgs. Likewise, the checker will automatically check scenarios in which messages are not delivered between host groups.

To simulate network duplication of messages, it is sufficient not to remove them from the msgs set.

### Broadcast
Most distributed algorithms use not just single point message sending, but *broadcast*.
* RequestVote and AppendEntries in Raft
* Prepare and Propose in Paxos

To simulate the sending of messages to all participants in the system, you can omit explicit recipients.

For example, in the Paxos spec, a single message addressed to all acceptors at once is placed in the network at the beginning of the first phase, with no addressee specified:

```
Send([type |-> "1a", bal |-> b])
```

Messages from acceptor to proposer need to specify addressee and sender. The sender is specified explicitly so that proposer understands when it will collect a quorum of responses from acceptors.

```
Send([type |-> "1b", acc |-> a, bal |-> m.bal,
      mbal |-> maxVBal[a], mval |-> maxVal[a]])
```

Such techniques simplify the algorithm compared to the real world, but at the same time reduce the granularity of events in the model, so there is a likelihood of losing race scenarios.

### Node failures
There are 3 main failure models:
* *Rejections* -- the node explodes and no longer accepts or sends messages.
* *Restarts* -- the node does not respond for some time, and then again starts participating in the algorithm.
* *Byzantine failures* -- the faulty node behaves in an absolutely arbitrary way - it came under the control of an attacker or violates the protocol due to an error in the implementation of the algorithm.

With the modeling of failures, everything is simple: we have chosen an asynchronous model, and in it, according to the FLP theorem, it is impossible to distinguish a dead node from the asynchronous network.

An algorithm that is robust to node restarts must store some of its state in a reliable persistent store.

For example, in RAFT, the replica in the leader selection phase must reliably save the cast vote before responding to the candidate's RequestVote request, otherwise, after restarting, it can vote in the same term a second time.

In Basic Paxos, the acceptor after Promise(b) in the first phase must ignore / reject messages with lower numbers. For this, the acceptor remembers the maximum ballot number that it received from the proposer. It is mandatory to remember this number in the persistent storage before sending a Promise / Accept, so that after a restart you will not forget about this promise.

In TLA+, you can simulate a node restart as a separate action, which resets its volatile state.

In the Raft spec, restarting a replica is modeled using the Restart action:

```
Restart(i) ==
    /\ state' = [state EXCEPT ![i] = Follower]
    /\ votesResponded' = [votesResponded EXCEPT ![i] = {}]
    /\ votesGranted' = [votesGranted EXCEPT ![i] = {}
    /\ voterLog' = [voterLog EXCEPT ![i] = [j \in {} |-> <<>>]
    /\ nextIndex' = [nextIndex EXCEPT ![i] = [j \in Server |-> 1]]
    /\ matchIndex' = [matchIndex EXCEPT ![i] = [j \in Server |-> 0]]
    /\ commitIndex' = [commitIndex EXCEPT ![i] = 0]
    /\ UNCHANGED <>
```


In this example, the persistent state (currentTerm, votedFor, log) is put into UNCHANCHED, the volatile (state, votesResponded, votesGranted) is reset to its initial values.

Modeling Byzantine failures requires a separate study and will not be discussed in this work.

### Timeout
Nodes in distributed systems use time to set timeouts to detect failures.

For example:

In Raft, the follower waits for a certain time (election timeout) for messages from the current leader, and if he does not receive them, he considers the current leader "dead", enters a new term and initiates the election of a new leader.

The Raft spec has a separate action for this timeout:

Timeout(i) == /\ state[i] \in {Follower, Candidate}
              /\ state' = [state EXCEPT ![i] = Candidate]
              /\ currentTerm' = [currentTerm EXCEPT ![i] = currentTerm[i] + 1]
              \* Most implementations would probably just set the local vote
              \* atomically, but messaging localhost for it is weaker.
              /\ votedFor' = [votedFor EXCEPT ![i] = Nil]
              /\ votesResponded' = [votesResponded EXCEPT ![i] = {}]
              /\ votesGranted'   = [votesGranted EXCEPT ![i] = {}]
              /\ voterLog'       = [voterLog EXCEPT ![i] = [j \in {} |-> <<>>]]
              /\ UNCHANGED <<messages, leaderVars, logVars>>

No clock is explicitly simulated.

### Non-determinism
Non-determinism is one of the main sources of complexity in the design of distributed systems. Non-determinism arises naturally due to the reactive nature of distributed systems: they react to external influences (client requests) and are subject to the influence of an uncontrolled environment (network, time flow), hardware/runtime (gc pauses) and failures.

TLA+ has two main ways of expressing non-determinism:
* Using the \E quantifier, you can choose which of the system nodes will "move" next: receive a message, reboot, etc.
* Using disjunction in actions: for example, a client can send either a SELECT or INSERT query to a database node:

    ClientAction == Insert \/ Select

Examples:

In the Raft spec, you select which message will be delivered or which of the nodes will perform the action

```
Next == /\ \/ \E i \in Server : Restart(i)
           \/ \E i \in Server : Timeout(i)
           \/ \E m \in DOMAIN messages : Receive(m)
           \/ \E m \in DOMAIN messages : DuplicateMessage(m)
```


In the Paxos spec, you select which message will be processed or which of the acceptors will respond to the request

Phase2b(a) == \E m \in msgs : /\ m.type = "2a"
                              /\ m.bal \geq maxBal[a]
                              /\ maxBal' = [maxBal EXCEPT ![a] = m.bal]
                              /\ maxVBal' = [maxVBal EXCEPT ![a] = m.bal]
                              /\ maxVal' = [maxVal EXCEPT ![a] = m.val]
                              /\ Send([type |-> "2b", acc |-> a,
                                       bal |-> m.bal, val |-> m.val])

Next == \/ \E b \in Ballot : \/ Phase1a(b)
                            \/ \E v \in Value : Phase2a(b, v)
       \/ \E a \in Acceptor : Phase1b(a) \/ Phase2b(a)

### System Properties
System / algorithm requirements are specified as properties. A property is a set of trajectories in the system state graph and are formulated as LTL formulas.

We will only be interested in 2 types of properties:
* *Safety-properties* - properties of the form []P, where P is an invariant for an individual state of the system. These properties say that "nothing bad" happens to the system.
* *Liveness properties* - properties of the form <>P, they say that something good eventually happens to the system.

When verifying distributed algorithms, it is enough to check only the properties of the two given types: any property can be expressed as the intersection of the safety and liveness properties (https://pron.github.io/posts/tlaplus_part3#safety-and-liveness)

Here are some examples of safety properties:

An example of an invariant in Raft: *Election Safety* in Raft -- no more than one leader can be selected in each term.
In TLA+, this property is expressed by negating the MoreThanOneLeader invariant

```
BothLeader( i, j ) ==
     /\ i /= j
     /\ currentTerm[i] = currentTerm[j]
     /\ state[i] = Leader
     /\ state[j] = Leader

MoreThanOneLeader ==
    \E i, j \in Server :  BothLeader( i, j )

OneLeader == [](~MoreThanOneLeader)
```

In Paxos, the safety property means that if the sentence (b, v) is selected (i.e. it was accepted by most acceptors), then any sentence with a higher number can only have the same value:

```
Agreed(v,b) == \E Q \in Quorum2: \A a \in Q: Sent2b(a,v,b)
NoFutureProposal(v,b) == \A v2 \in Value: \A b2 \in Ballot: (b2 > b /\ Sent2a(v2,b2)) => v=v2

SafeValue == \A v \in Value: \A b \in Ballot: Agreed(v,b) => NoFutureProposal(v,b)

OnlyOneValue == []SafeValue
```

When checking this invariant, we use the fact that messages from the set msgs are not deleted, which means that in each of the states, all selected sentences can be restored from it.


In Kafka, appends that are verified by the leader must be in the logs of all quorum replicas. In the TLA+ spec, this is the StrongIsr property:

```
StrongIsr == \A r1 \in Replicas :
    \/ ~ ReplicaPresumesLeadership(r1)
    \/ LET  hw == replicaState[r1].hw
       IN   \/ hw = 0
            \/ \A r2 \in quorumState.isr, offset \in 0 .. (hw - 1) : \E record \in LogRecords :
                /\ ReplicaLog!HasEntry(r1, record, offset)
                /\ ReplicaLog!HasEntry(r2, record, offset)

Safety == []StrongIsr
```

The WeakIsr property differs in that it checks that the record is present for all replicas that the current one considers to be its ISR.

We will not consider the liveness properties:

Replication in real distributed systems is usually implemented by solving the distributed consensus problem, and in the asynchronous model, it is impossible to ensure the termination of consensus due to the FLP theorem.

If we abandon the consensus, then we can achieve the termination of the algorithms, but at the same time we would have to weaken the consistency model to a weak guarantee for the user, but we do not consider it.

### Checking consistency models
The previous paragraph gave examples of safety properties. If we look at the system from the outside rather than from the inside, all algorithms are hidden from us, we only have access to observable behavior, which is formulated in terms of consistency models.

There is a whole range [https://jepsen.io/consistency] of consistency models for distributed storage. But the most common are two models: *eventual consistency* and *linearizability*. We are primarily interested in the strongest consistency model - linearizability, since it is the most understandable to the user. There is already a ready-made spec to check linearizability https://github.com/lorin/tla-linearizability.
