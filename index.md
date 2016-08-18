---
type: homepage
toc: false
---

GigaPaxos is a group-scalable replicated state machine (RSM) system, i.e., it allows applications to easily create and manage a very large number of separate RSMs. Clients can associate each service with a separate RSM of its own from a subset of a pool of server machines. Thus, different services may be replicated on different sets of machine in accordance with their fault tolerance or performance requirements. 

The underlying consensus protocol for each RSM is Paxos, however it is carefully engineered so as to be extremely lightweight and fast. For example, each RSM uses only ~300 bytes of memory when it is idle (i.e., not actively processing requests), so commodity machines can participate in millions of different RSMs. When actively processing requests, the message overhead per request is similar to Paxos, but automatic batching of requests and Paxos messages significantly improves the throughput by reducing message overhead, especially when the number of different RSM groups is small, for example, gigapaxos achieves a small-noop-request throughput of roughly 80K/s per core (and proportionally more on multicore) on commodity machines. 

The lightweight API for creating and interacting with  different RSMs allows applications to “carelessly” create consensus groups on the fly for even small shared objects, e.g. a simple counter or a lightweight stateful servlet. GigaPaxos also has extensive support for reconfiguration, i.e., the membership of different RSMs can be programmatically changed by applications by writing their own policy classes.

The [GigaPaxos paper](http://www.cs.umass.edu/~arun/papers/gigapaxos.pdf) talks more about GigaPaxos in greater detail. 

To get started with GigaPaxos checkout the [Getting Started]({{ site.baseurl }}/documentation/) page.
