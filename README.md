# The Chubby lock service for loosely-coupled distributed systems

## Introduction

Intended to provide coarse-grained locking as well as reliable storage for a
loosely-coupled distributed system. Allow clients to synchronize their activities and to agree on basic information about their environment.

Main goals is reliability and availability. Second: throughput and storage capacity.

* Google File System uses a Chubby lock to appoint a GFS master server.
* Bigtable uses Chubby to elect a master, to allow the master to discover the servers it controls and to permin clients to find the master.
* Both GFS and Bigtable use Chubby as available location to store a small amount of meta-data; store the root of their distributed data structures.

## Design

Why not build a library implementing Paxos and instead create a centralized lock service?
  * Most programs start as prototypes and aren't designed for high availability; invariably the code has not been specially structured for use with a consensus protocol.
    A lock server makes it easier to maintain existing program structure and communication patterns.

Usage example for GFS: elect master to write to an existing file server. One would acquire a lock to become master,
pass an additional integer (the lock acquisition count) with the write RPC, and add an if-statement to the file server
to reject the write if the acquisition count is lower than the current value.

Distributed-consensus algorithms use quorums to make decisions, so they use several replicas to achieve high availability.
For example, Chubby itself usually has five replicas in each cell, of which three must by running for the cell to be up.

Because of the above following was chosen:
  * A lock service, as opposed to a library or service for consensus
  * Serve small-files to permit elected primaries to advertise themselves and their parameters, rather than build a maintain a second service.
  * A service advertising its primary via a Chubby file may have thousands of clients. Therefore we must allow thousands of clients to observe this file.
  * Implement event notification mechanism
  * Caching is desirable if there are clients polling
  * Use consistent caching
  * Provide security mechanisms
  * Coarse-grained locking is expected, instead of fine-grained. This means that lock-acquisition rate is only weakly related to the transaction rate of the client applications. Temporary lock server unavailability delays clients less. However, the transfer of a lock from client to client may require costly recovery procedures. Thus it's good, if a fail-over of a lock server would not cause locks to be lost.

## System structure

Chubby consists of a server and a library the client applications link against and communication takes place via RPC.
A Chubby cell consists of a small set of servers known as replicas, placed so a to reduce the likelihood of correlated failure.
Replicas use distributed consensus protocol to elect a master. The master must obtain votes from a majority of the replicas,
plus promises that those replicas will not elect a different master for an interval of a few seconds known as the master lease.
The replicas maintain copies of a simple database, but only the master initiates reads and writes of this database.

## Files, directories and handles

Chubby exports a file system interface simpler than UNIX. A typical name: `/ls/foo/wombat/pouch`.
The ls prefix stands for lock service. foo is the name of a Chubby cell; it is resolved to one or more Chubby
servers via DNS lookup. It's different from UNIX in a number of ways:
* cannot move files from one directory to another, because different directories can be served from different Chubby masters
* do not maintain directory modified times
* permissions are controlled on the file itself and does not depend on the path leading to the file
* no symbolic or hard links

Node is collective name for files and directories. They may be permanent of ephemeral.
Ephemeral files are used as temporary files, and as indicators to others that a client is alive.

## Locks and sequencers

 Advisory locks are used instead of mandatory for a number of reasons.

 A lock holder may request a sequencer, an opaque byte-string that describes the state of the lock immediately after acquisition. It contains the name of the lock, the mode in which it was acquired (exclusive or shared) and the lock generation number. The client passes the sequencer to servers if it expects the operation to be protected by the lock. The recipient server is expected to test whether the sequencer is still valid.

## Events

When a file handle is created a client can subscribe to a range of events. Messages are delivered asynchronously.
* file contents modified -- often used to monitor the location of a service advertised via the file
* child node added, removed, modified
* Chubby master failed over
* handle has become invalid
* lock acquired -- can be used to determine when a primary has been elected
* conflicting lock request from another client

## API

Similar to UNIX. Partial reads and writes are not allowed and they are atomic.

Primary election is performed as follows:
All potential primaries open the lock file and attempt to acquire the lock. One succeeds and becomes the primary, while the others act as replicas. The primary writes its identity into the lock file with `SetContents()` so that it can be found by clients and replicas, which read the file with `GetContentsAndStat()`,
perhaps in response to a file-modification event.

## Caching

Chubby clients cache file data and node meta-data in a consistent, write-through cache held in memory. The cache is maintained by a lease mechanism, and kept consistent by invalidations sent by the master, which keeps a list of what each client may be caching.

## Sessions and KeepAlives

Chubby session is a relationship between a Chubby cell and a Chubby client. It is maintained by periodic handshakes called KeepAlives.
Client is forced to acknowledge cache invalidations to maintain the session. Event notification is passed in the KeepAlive messages from the server.

If a client's local lease timeout expires, it becomes unsure whether the master has terminated its session. The client empties and disables its cache, and we say that its session is in jeopardy. The client waits a further interval called the grace period, 45s by default. If the client and master manage to exchange a successful KeepAlive before the end of the client's grace period, the client enables its cache once more.

## Fail-overs

Super long fucking process. In principle: master dies, all client sessions are in jeopardy. Hopefully, a new master is elected before the grace period has ended for all the clients. KeepAlive messages get to the new master and it informs all the clients that they need to use a new client epoch number, because he's a new master. Then it starts recreating the state of the previous master by performing database queries, contacting clients for information (hell knows why). Tells all the clients to invalidate their caches and waits either for them to acknowledge this or lets their sessions expire. And tries to make sure ephemeral files are in order. It's executed rarely and of course has lots of bugs.
