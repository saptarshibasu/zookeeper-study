# Zookeeper Study

- [Background Concepts](#background-concepts)

  - [Linearizability](#linearizability)

  - [Total Order](#total-order)

  - [Atomic Broadcast And Concensus](#atomic-broadcast-and-concensus)

- [What is Zookeeper](#what-is-zookeeper)

- [Zookeeper Primitives](#zookeeper-primitives)

- [How Zookeeper Works](#how-zookeeper-works)

  - [Client Sessions](#client-sessions)

  - [Reads And Writes](#reads-and-writes)

  - [Leader Election](#leader-election)

  - [Recovery](#recovery)

  - [Notifications](#notifications)

- [References](#references)


## Background Concepts

### Linearizability

* A system is linearizable if it behaves as if it holds a single copy of data that changes are made atomically
* It is called a recency guarantee: A read is guaranteed to see the latest value written
* If after a write operation, a read operation returns the new changed value, all subsequent reads will return the same new value until it is overwritten again
* Co-ordination services like Zookeeper and etcd use consensus algorithms to implement linearizability
* Many systems do not implement lineraizability because lineraizability is slow
* It has been proved that if we want a linearizable system, the response time of read and write requests is at least proportional to the uncertainty of delays in the network

* **Use Cases**
  * Locks and Leader Election - Locks are often used to elect leader in a distributed system. The process that acquires the lock becomes the leader. Here the lock has to be linearizable, so that once a process P acquires it, all other processes agree that the lock is already acquired by A
  * Constrainst & Uniqueness Guarantees - Airlines cannot sell more seats than available, one cannot withdraw more money that available in the bank account, two users cannot be registered with the same username

### Total Order

* A total order allows elements to be compareable. So, if there are two elements, we can say one is greater than the other. Hence natural numbers (1,2,3,4) are totally ordered
* However, mathematical sets like ({a,b}, {b,c}) are incomparable and hence partially ordered
* In a linearizable system, we have a total order of operations: if the system behaves as if there is only a single copy of the data, and every operation is atomic, this means that for any two operations we can always say which one happened first. There is a single timelie on which all operations are totally ordered
* Similrly, if we have total order broadcast, we can build linearizable storage on top of it
* A sequence of operations or requests in a distributed system can be totally ordered by assigning them a totally ordered sequence number from a monotonically increasing counter
* Total Order Safety properies - 
  * Reliable delivery - No messages are lost: if a message is delivered to one node, it is delivered to all nodes
  * Totally ordered delivery - Messages are delivered to every node in the same order
* However, this total order alone cannot guarantee uniqueness constraints like unique username because when a node has received a username creation request, it doesn't know if another node is concurrently in the process of creating the same username with a lower sequence number

### Atomic Broadcast And Concensus

* An Atomic Broadcast algorithm ensures that the total order properties are met even if one or more nodes are faulty
* An important aspect of total order broadcast is that the order is fixed at the time the messages are delivered: a node is not allowed to retroactively insert a message into an earlier position in the order if subsequent messages have already been delivered
* It can be proved that a linearizable compare-and-set (or increment-and-get) register and total order broadcast are both equivalent to consensus. That is, if you can solve one of these problems, you can transform it into a solution for the others

## What is Zookeeper

* Zookeeper is a service for co-ordinating processes of a distributed system
* The ZooKeeper service comprises an ensemble of servers
* Write requests are processed by an agreement protocol known as Zookeeper Atomic Broadcast (ZAB) - an implementation of Atomic Broadcast
* ZAB requires a quorum (majority) of servers to work. Therefore the ensemble should consist of an odd numebr of servers
* Thus in an ensemble of 5 servers, a minimum of 3 servers is required to be up and running for Zookeeper to function. Zab works when a majority of the servers or quorum are correct. It can tolerate `f` failure in an ensemble of `2f + 1` servers
* At the core of Zookeeper is a linearizable storage service

## Zookeeper Primitives

* ZooKeeper provides to its clients the abstraction of a set of data nodes (`znodes`), organized according to a hierarchical name space
* Clients can create, delete or read a `znode` value
* `znode` an be either `persistent` (persists until client explicitly deletes it) or `ephemeral` (gets deleted if the client session terminates or the client explicitly deletes it)
* Every transaction is associated with a relative sequence number called `zxid`
* Clients maintain a connection with a Zookeeper server and sends periodic heartbeats. If the server crashes or the network is partitioned, the client will try to connect to a different server. As long as the client can connect to a different server within a timeout, the `session` will be maintained
* A `znode` can also be set to be `sequential` (both persistent and ephemeral). A sequential `znode` is assigned a unique, monotonically increasing integer. This sequence number is appended to the path used to create the `znode`
* Clients can register with ZooKeeper to receive notifications of changes to specific `znodes`. Registering to receive a notification for a given `znode` consists of setting a `watch`
* A `watch` is a one-shot operation, which means that it triggers one notification. To receive multiple notifications over time, the client must set a new `watch` upon receiving each notification
* Every `znode` is associated with a `version` that gets incremented with every write operation
* Some write operations take the `version` as the input and if the input `version` does not match with the `znode` current version, the operation will fail to avoid race conditions between multiple Zookeeper clients

## How Zookeeper Works

### Client Sessions

* Every Zookeeper server services clients
* Clients connect to exactly one server to submit its requests
* If the server does not hear from a client within session timeout (`t`), the server declares the client session expired
* If the client doesn't hear from the server within 1/3 of `t`, it sends a heartbeat
* If the server doesn't respond, within 2/3 of `t`, the cleint starts looking for a different server in the ensemble to prevent the session from timing out
* If the last `zxid` seen by a server is `i`, and the client has already seen a `zxid` `j`, where `i < j`, the client cannot connect to the given server

### Reads & Writes

* Read requests are serviced from the local replica of each server database
* Requests that change the state of the service, write requests, are processed by an agreement protocol known as Zookeeper Atomic Broadcast (ZAB) - an implementation of Atomic Broadcast
  * Write requests are forwarded to the single leader node of the ensemble
  * The rest of the nodes - the followers - receive message proposals from the leader
  * The followers send acknowledgement to the leader
  * Upon receiving acknowledgements from a quorum of servers, the leader asks the followers to commit the transactions
* As the read requests are served by the server from its local in-memory state, it provides good read performance, but at the same time the returned value could be stale
* Every read response from a server is tagged with a `zxid` that the server has last seen
* If the client needs the latest value, it can issue a `sync` request before the read operation. This will ensure that all the latest changes upto the instant the `sync` has been issued are replicated to the follower serving the read request

### Leader Election

* Zab relies on the fact that in any two quorums in the ensemble, there is at least one overlapping node
* A leaders broadcasts proposals with an epoch number, say `e`
* At the time of election, the quorum members promise that going forward they will not accept any proposal with epoch `e-1`
* Thus if an existing leader crashes, the new leader needs to ensure that it has all transactions of the previous leader (epoch `e-1`). As the new leader has already got the support of a quorum to get elected as a leader and a quorum will always have at least one overlapping member from the previous epoch (quorum), the new leader can get all the transactions of the previous leader
* The new leader ensures that it has all the transactions of the previous epochs replicated in its log, before proposing new transactions

### Recovery

* Writes are forced to be on disk before they are applied to the in-memory database
* ZooKeeper provides high availability by replicating the ZooKeeper data on each server that composes the service
* Zookeeper nodes maintain a write ahead log (WAL) containing an ordered list of idempotent transactions
* Each Zookeeper node also applies these idempotent transations to its in-memory database
* The leader transforms the client request into an idempotent request i.e. the future state is calculated and added to log, applied to the in-memory state and sent to the followers 
* To do a recovery from a crash, the in-memory database can be recreated by applying all the trasactions in the log. However, it may take a prohibitively long time depending on the number of transactions. Therefore, Zookeeper nodes take periodic snapshots of the in-memory state. This helps to recover the in-memory state quickly by re-applying only the delta transactions on top of the last snapshot
* The snapshot does not represent a single point in time and it may contain some transactions that happened while the snapshot taing process is going on. However, as the transactions in the log are all idempotent, if a transaction is applied more than once, it won't cause any problem

### Notifications

* When a server processes a write request, it also sends out and clears notifications relative to any watch that corresponds to that update
* Servers process writes in order and do not process other writes or reads concurrently
* This ensures strict succession of notifications
* servers handle notifications locally. Only the server that a client is connected to tracks and triggers notifications for that client

## References

* Kleppmann, Martin. Designing Data-Intensive Applications (Kindle Location 9571). O'Reilly Media. Kindle Edition. 
* Junqueira, Flavio. ZooKeeper: Distributed Process Coordination . O'Reilly Media. Kindle Edition. 
* https://www.usenix.org/legacy/events/atc10/tech/full_papers/Hunt.pdf



