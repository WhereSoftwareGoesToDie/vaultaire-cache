Zookeeper vs Etcd
=====================

To store the current state of the distributed cache, we need to store, who has which data. There are multiple proticols and implementations thereof to adress this Problem. The most prominent protocols are [Paxos](http://en.wikipedia.org/wiki/Paxos_(computer_science)) and [Raft](http://en.wikipedia.org/wiki/Paxos_(computer_science)).

Notable implementations are:

- Zookeeper
- Etcd
- Doozer
- Kontiki (Haskell)

*Kontiki* is still in a very early stage and therefore not stable enough.
The other implementations are nicely summed up by <http://devo.ps/blog/zookeeper-vs-doozer-vs-etcd/>.

With *Doozer* being discontinued, *Zookeeper* and *Etcd* are possible choices.

Functionality
---------------
Both Zookeeper and Etcd offer a globally consistent key-value store (CREATE, SET, DELETE, LIST). Zookeeper has some additional functionality however:

- Ephemeral nodes. These are linked to a specific client connection and get deleted when the client disconnects.
- Sequence Flag. When inserting a new node with this flag, it will be appended by a unique monotonically increasing sequence number.
- Watchers. Each node/directory can be watched. A watcher is a callback function which is calles when something changed.

Performance
-------------
I ran a benchmark creating, setting and deleting nodes with multiple (10 each) threads concurrently. Scrips and results can be fount at:

- Etcd: <https://gist.github.com/tvh/7bc680ad357b63b93247>
- Zookeeper: <https://gist.github.com/tvh/f939d90306e55145a7e8>

This is the throughput per thread:

|          | Etcd | Zookeeper (same machine, no replication) | Zookeeper (3 nodes) |
|----------|------|------------------------------------------|---------------------|
| create/s | 0.9  | 50                                       | 5.2                 |
| set/s    | 1.9  | 110                                      | 11                  |
| delete/s | 1.9  | 125                                      | 10                  |

If the number of threads go up, the throughput per thread drops only marginally. This is in line with the initial results presented in the paper: <https://www.usenix.org/event/usenix10/tech/full_papers/Hunt.pdf>

