# Gossip: a good foundation for distributed systems

A cornerstone component of distributed systems is the failure detector. Yet, in the real world,
the failure of a device on a realistic network is impossible to detect with complete reliability and accuracy [^1].

To achieve distributed consensus, software engineers design for a world without a perfectly reliable failure detector. 
We've gotten really good at using merely "okay" failure detectors. Algorithms like Paxos ([made simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf))
are written to be correct even in the absence of a perfect failure detector.

## the need for failure detection

In fact, we can write a really simple consensus algorithm that works even in the presence of failure: Simply do nothing and keep the initial value.
This trivial example is a perfect algorithm for achieving distributed consensus! Of course, it is not useful. We want algorithms that can make progress
when it is possible to do so safely. While it is not possible to always guarantee progress if any device can fail, Paxos provides a sketch algorithm
that comes very close! At the heart of the algorithm is the snyod protocol, which is very safe: once the alogirithm "finishes"[^2], the participating devices
will have chosen a single value (although only some of those devices may know that).

As a note for people who already know what we're talking about here, for this context we will assume that devices only fail by crashing. In other words,
we have no malicious or otherwise misbehaving devices. We're not talking about blockchain (or more academically, "byzantine fault tolerance").

However, as part of its safety – the snoyd protocol can be interrupted. Any device _can attempt_ to take over "leadership" of a round of the snyod protocol.
When this happens, nothing can be done but either allowing that device to win (potentially creating an infinite loop) or telling that device to step down (sacrificing
availability if the previous leader fails).

> ### a deeper look at the snoyd protocol
> 
> coming soon

So how do we improve this protocol? Well, if we just have one node talk attempt to lead the group at a time, then the group will make progress when it is
possible for the leader to communicate with a quorum of the group. As we have a stable leader, the group can make progress.

Here is the magic intuition of Paxos: if we have a sometimes-reliable failure detection mechanism, then we can make progress whenever the mechanism is working.
And, if the mechanism fails, we do not loose safety! We just loose the ability to make progress.

Many good distributed systems work on some variation of Paxos[^3]. And this crucial component, the failure detector, is important.

## gossip protocols

a another kind of distributed systems algorithm class is gossip/group-membership algorithm. Gossip algorithms have some parallels with consensus algorithms: they both attempt
to share information between a group of devices. But while consensus algorithms try to syncronize every device to come to a consistent view (sort of acting as a single unit),
gossip algorithms are okay with inconsistency and merely attempt to share information with every other nodes in a population. Gossip algorithms are effective for creating
membership detectors. For example, [SWIM](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf). We can discover nodes who are part of a population in a fault-
tolerant way, and even add new nodes to the system without any centralized leader. We can also detect and share information about node failure.

## multi-raft and failure detecting optimizations

modern implementations of consenus alogirithms (like [tikv's](https://tikv.org/deep-dive/scalability/multi-raft/) and [CockroachDB](https://www.cockroachlabs.com/blog/scaling-raft/))
have found an optimization for raft: have each device be a member of many groups. by doing so, we can achieve better per-node concurrency and more effectively take advantage of
the inherent paralleism of modern hardware. we also can put the membership detection responsibility on the device, rather than have a seperate detector for every cluster a device is a member of.

## using gossip-based membership protocls!

we can use gossip for this, too! gossip can provide a way for forming groups of devices to cooperate with a consensus algorithm, and then scalably detect failures. a good failure
detection mechanism may even help reduce transient failures which can have disastrous availability consequences [in the real-world](https://blog.cloudflare.com/a-byzantine-failure-in-the-real-world/).
plus, we can use things like [network coordinates](https://sites.cs.ucsb.edu/~ravenben/classes/276/papers/vivaldi-sigcomm04.pdf) (inspired by [Serf](https://www.serf.io/docs/internals/coordinates.html))
to adapt group membership based on the state of the network as delays between nodes change (as they often do in real life).



[^1]: given by the FLP impossibility therom: https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf
[^2]: word used remarkably informally
[^3]: the phrase "variation of Paxos" is... controversial? but algorithms like raft are similar enough. See https://dl.acm.org/doi/pdf/10.1145/3380787.3393681, one of my favorite papers on the topic.
