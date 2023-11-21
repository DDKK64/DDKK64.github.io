---
title: '6.824 - ZooKeeper'
layout: posts
---
# ZooKeeper

Paper: Wait-free coordination for Internet-scale systems

Provides primitives for coordination service. (The paper calls coordination kernl)

High performance at the cost of weaker but carefully designed consistency.

## Data model

Hierarchically organized namespace:

![](/assets/images/courses/6.824/reading/zk-znodes.jpeg)

Znodes are identified by UNIX path names.

Type of znodes:

- Regular znodes: created and deleted explicitly.
- Ephemeral znodes: automatically deleted when the session that creates it terminates.

Watch machanism: clients receive notifications of changes from servers without polling

## API

`create(path, data, flags)`

- sequential flag: append monotonizally increased value to the node name, which indicates creation order and avoids name collisions.

`delete(path, version)`

`exists(path, watch)`

`getData(path, watch)`

`setData(path, data, version)`

`getChildren(path, watch)`

`sync()`: wait for all currently pending changes to propagate

All functions support both synchronous and asynchronous versions.

Writes are executed only if `version` matches the znode version or if `version` is -1, so that version check is skipped.

## Guarantees

- Linearizable writes: write operations are assigned total order and respect precedence
- FIFO client order: all request from a client are executed by sending order.
  - Writes in the client order
  - Read returns the last write of the same client 
- Notifications are deliverd to the client in the original mutation order.
- Slow read: a read after a sync. sync flushes pending updates and read the latest value up to sync. (coordinate communication outside ZK)
- Durability: replies success to the client after persistence.

> Linearizability: 'behaves like a single mahcine'
> 1. Total order of operations
> 2. Order matches real time
> 3. Read always returns the last write

Does not guarantee full linearizability. E.g. a client may read stale data after some other client write the new data.

*A-linearizability (asynchronous linearizability)*

### Examples


## Primitives

### Test-and-Set

Work with the version number.

### Dynamic configuration

Processes watches a common znode for configuration. After a notification, they reads the data.

### Group membership

For a group creates a znode.

Each process in the group creates a a ephemeral znode with its own unique id or with the sequential flag set.
When a process ends, its znode is automatically removed.

Each process can get group information by listing/watching the children of the group znode.

### Fair exclusive locks 

Lock is aquired by request order.

Lock is automatically released from failed client.

```text
Lock():
    n = create(lockName + "/lock-", EPHEMERAL | SEQUENTIAL)
    while true:
        C = getChildren(lockName, false)
        if n is the first node in C:
            return n
        p = the znode in C which immediately precedes n
        exist(p, true)

Unlock():
    delete(n)
```

In `Lock()`, why loop again after p is removed: p may be removed before its preceding znodes (due to failure). 

### Read/Write locks

```text
WriteLock():
    n = create(lockName + "/write-", EPHEMERAL | SEQUENTIAL)
    while true:
        C = getChildren(lockName, false)
        if n is the first node in C:
            return n
        p = the znode in C which immediately precedes n
        exist(p, true)

ReadLock():
    n = create(lockName + "/read-", EPHEMERAL | SEQUENTIAL)
    while true:
        C = getChildren(lockName, false)
        if all znodes in C up to n are all read mode:
            return n
        p = the last write znode which precedes n
        exist(p, true)

Unlock():
    delete(n)
```

TODO: R/W Lock conversion

### Double Barrier

Synchrinize the beginning and the end of an execution.

```text
Enter():
    n = create(barrierName + "/ready-", EPHEMERAL | SEQUENTIAL)
    while C = getChildren(barrierName, true):
        if the length of C reaches up to threashold:
            return n

Leave(n):
    delete(n)
    while C = getChildren(barrierName, true):
        if the length of C reaches down to threashold:
            return
```

## Implementation

Each write request is transformed to a indemponent transaction.

Atomic broadcast (consensus) by ZaB, based on quorum-based, communicate over TCP, persistent with WAL

Fuzzy snapshot: only requires atomic read of each znode. The consistency within snapshot does not matter, since changes are indemponent and can be safely replayed.

`sync` operation is indeed a null write that causes the server to catches up with leader.

### Client-Server iteractions

Write requests: all writes are forwarded to the leader in batches. (lower the average transmission overhead).

Notification: updates at local replica trigger notifications to be sent to conneted clients in order.

Read requests: locally handled at replicas. Clients may read stale values. Using `sync` to flush pending writes if necessary.

Heartbeat messages are sent from clients during idle time.

FIFO order are guarranteed by using `zxid` (similar to the log index of raft):

- Client keeps the newest zxid it received from zookeeper
- zxid are tagged by the server on response and heartbeat 
- Client send requests with its zxid and server responds only if it has greater or equal zxid.

Client reconnect: when a client connects to a new server, the server guarantees to establish session only after it catchs up with the zxid of the client.

Client failure: the leader determines client failure if no server receives any message from the client within session timeout. 

When the connected server failed, timely reconnection may avoid session timeout.

## Performance

- Throughput
- Zab performance
- Failure recorvery
- Latency
- Application performance

![](/assets/images/courses/6.824/reading/zk-fig5.png)