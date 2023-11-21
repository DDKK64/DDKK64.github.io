# Distributed transactions

Cross-machine atomic opperations

## Before-or-after atomicity

Before-or-after atomicity: concurrent actions complete *as if* one is either completely before or after another.

Correctness of before-or-after atomicity: coordination is correct the result can be obtained by some serial schedule of actions.

## Concurrency control

Pessimistic: lock

Optimistic: lock-free. Abort if not serializable.

## Two-phase locking  (2PL)

Phase I: A transaction aqcuires locks as it proceeds but never releases any before the lock point.

Lock point: the instant when the transaction obtains all the locks it needs.

Phase II: The transaction releases locks it holds but never aqcuires any.

## Distributed two-phase commit

Multi-site transaction: a transaction that executes component transactions at multiple sites on top of a best-effort network.

Requirements for multi-site transaction:

- Exactly-once-execution RPC
- Support for single-site transaction

Phase I: the time before all workers send PREPARED messages.

1. The cooridnator sends component transactions to the workers. Workers may log the precommit results, e.g. snapshot write?.
2. The coordinator persistently sends PREPARE messages.
3. Workers persistently sends PREPARED.
4. The coordinator waits for all workers' PREPARED messages.

Phase II: workers commit their parts of transactions.

1. The coordinator logs the transaction outcome (COMMITED or ABORTED). Then it sends COMMIT messages to workers.

![](/assets/images/courses/6.824/reading/two-phase.jpeg)

Worker cannot revert its prepared state after the PREPARED message is sent, even after a crash.

Variations and optimizations:

- Coordinator waits for fourth ack messages from worker sites. After all acks are received it drops the outcome record.
- Presumed commit

[two general's problem] No way to guarantee that workers do their transactions simultaneously.

Raft v.s. 2PC:

- Raft helps to reach consensus about the same thing
- 2PC distributes different tasks across servers

## Two generals' problem

No protocol with bounded messages can convince both generals that it is safe to match.