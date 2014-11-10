Vaultaire Cache
===============

This document sketches the design of a new service to cache and answer queries
about data in the Vaultaire data store. This new service will reduce load on
Vaultaire, reduce latency of responses, and provide an obvious place to
implement additional functionality such as indexing and aggregation.

Requirements
------------

- Scalable and available.

- Configurable replication of data (from none to arbitrary fixed number of replicas).

Components
----------

1. Broker - receives requests from clients and distributes to workers; receives
telemetry from workers and aggregates.

2. Zookeeper - maintains consistent maps of data locations, replicas, and load
operations.

3. Workers - maintain caches of data and answer queries.

Architecture
------------

Data is managed as individual time series divided into pieces of a fixed size.
We will call each such piece a *chunk* (e.g. the load average of machine
X between $time_{1}$ and $time_{2}$) and assume that each chunk has a unique
identifier (e.g. the CRC32 hash of the address and timestamps).

At any point in time each *chunk* will be cached in memory by zero or more
workers. Each worker maintains a location map which describes which, if any,
workers have each *chunk* loaded. The protocol used to maintain the this map is
described below.

Client requests are received by the broker and queued for processing by workers
as they become available.

When processing a request, a worker will inspect it determine which *chunks*
will be required to answer the query. If the worker has the necessary data it
will answer the query; if not, it will lookup the worker/s which do have the
required data and forward the request (if there are multiple candidates, one
will be chosen at random).

When forwarding a request, a worker will set a short timeout on the ZMQ socket;
if the timeout expires before receiving an acknowledgement, it will instead
fetch a copy of the data from a worker which already has the data and answer
the query itself.

Location Map
------------

The location map is a key/value store which associates a *chunk* identifier
with a list of nodes which currently have that *chunk* in memory.

It is unclear whether the map should be globally consistent. If so, the map
will be maintained in Zookeeper (with ephemeral znodes used to prevent problems
with stale entries). If the map proves too large for Zookeeper to store, it can
be used to maintain and propagate changes to the cluster state (e.g. a leader
who has an up to date map and a log of map updates). An alternative approach
might replace Zookeeper with `etcd`, or make use of an internal consistency
module like `kontiki`.

If consistency is *not* required, the system might use a gossip protocol to
distribute updates or even just broadcast announcements via, e.g., a pub/sub
zeromq socket managed by the broker.

Duplication and replication
---------------------------

It is unclear whether the system requires *replication* of data (i.e. a strong
lower bound on the number of copies of a piece of data) or just *duplication*
(i.e. one or more copies created when required to address the workload).

If replication is required, the system will maintain a replication map similar
in structure to the location map. Workers will listen for updates to this map
(with Zookeeper) or the absence of heartbeat messages (with gossip or broadcast
announcements) to detect the disappearance of replicas. When such an event is
detected, a node will attempt to nominate itself as the replacement replica
(re-create the deleted emphemeral znode in Zookeeper). If successful, it will
fetch a copy of the data and add itself to the location map.

If only duplication is required, a worker may elect to maintain a duplicate of
some *chunk* when a request it forwards to another node times out. Some
experimentation and tuning will be required to ensure that this does not result
in churning of *chunks*.

