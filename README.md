# notes on anti-fragile systems

collection of ideas for building anti-fragile systems (distributed systems that tolerate failures and self-heal).

ideally, we can get to a place where we can replace [remote key-value stores](https://pages.cs.wisc.edu/~rgrandl/papers/link.pdf)
and our reliance on the 12-factor app. would be cool if we can keep application state in-memory without loosing horizontal scalability
or drastically increasing complexity of application development. 

this is, of course, not straightforward to do! but hopefully by creating strong components for distributed systems, we can get there.

possible components:

- [gossip-based membership detection](/notes/gossip.md)
- scalable consensus mechanism, with support for rolling updates of nodes. one proposal: [deconstructed raft](/notes/log-storage.md)
- first-class support for [network simulation](https://sled.rs/simulation.html)
- automatic consensus group formation & healing based on gossip and network coordinates
- a database engine native to the replicated state machine abstraction [notes](/notes/replicated-log-structuring.md).
- parallelizable actors-model concurrency backed by raft state machines. by seperating "query" message processing from "command" state updates,
  should be possible to get great horizontal scalability

all of these notes are currently in draft form... i'm following a practice of "work in public", but that does mean this work is probably very
bad at first! there are likely to be mistakes and ommissions, and at present I may be missing relevant work available in the research community.
