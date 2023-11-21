---
layout: posts
title: '6.824 - Frangipani'
---
# Frangipani

Paper: Frangipani: A Scalable Distributed File System. 1997

A network file system aimed for data sharing among workstations.

- Cache coherence
- Distributed locking
- Distributed fault recovery

## Users' view

Researchers work on trusted servers: coding, compiling, editing etc.

Users access the file system with the UNIX file API. 

Some files are be shared.

Design choices:

- Write-back caching
- Strong consistency
- Low latency

## Architecture

![](/assets/images/courses/6.824/reading/frangipani-fig2.png)

File system is embeded at the client (the Frangipani server), registered in the kernel.

## Disk layout

![](/assets/images/courses/6.824/reading/frangipani-fig4.png)

Petal: a fault-tolerant virtual disk. Provides $2^{64}$ bytes of adress space.

Physical disk space are allocated on write.

Each server exclusively holds a portion of logs space. There're 256 portions in the figure.

## Lock service

Read-write lock:

- Read lock allows read and cache. Release of read lock invalidates the cache.
- Write lock allows read, write and cache. Downgrade of the lock flushes dirty data. Release of the lock flushes dirty data and invalidates the cache.

Locks are sticky: a client does not release a lock until it is asked to do so.

Liveness of a Frangipani server are detected using lease. It must renew its lease periodically. 
After timeout, it is considered to have failed. Then the recovery procedure begins, during which the logs are replayed and the locks are released.

Locks are organized with lock tables distributed across Fragipani servers.
Each table holds distinct sets of locks.
The assignment of the tables are determined by the lock servers.

Lock servers are replicated with a consensus algorithm.

Deadlock avoidance: the server sorts locks by inode address and aquires them in order.

## Logging

Reminiscent of the journalling file system. Basic assumption to achieve atomicity: disk sector write is atomic.

Changes of metadata (e.g. directory, timestamps etc.) are logged.

File content changes are not logged.
Justification for not logging the file contents: even in a local UNIX filx system, the file writes may fail halfway and leaves the file in a consistent state.

WAL: before performing an update, the server logs the update first.

Log is structured as a circular buffer. Each sector is attached with an monotonically increasing sequence number. The beginning/end of the log is identified by the sequence number.

A single log record can span multiple sectors. Checksum is used to detect incomplete record.

In the presence of multiple logs, how to determine the correct serialization

- Each log record is associated with a version number obtained from locks service.
- Each metadata sector is attached with a version number of the corresponding log.
- The log is replayed only if the log has a newer version number than the metadta block.

Logs are first written in memory, then periodically flushed to Petal.
Synchrony can be achived at the expanse of higher latency.

## Failure

Crash before the log is persisted: changes are lost.

Crash after the log is persisted:

1. If the lease expire, then the server is considered to fail. To deal with network partition:
   - The recovery daemon wait for the lease to expire before taking recovery procedure.
   - The server never writes to pedal or aquire any new locks after the expiration.
2. The recovery daemon take the ownership of the locks and the log of the crashed server and starts replay.
