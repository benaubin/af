# Log Storage ideas for raft

The [Raft paper][raft] and original [Paxos paper][paxos] suggest that each device in a consensus group have attached stable storage. Later discussions of Paxos
explore how this idea works in practice, such as [Disk paxos]. The same depth of exploration has not happened for Raft.

Anyways... many modern cloud systems are comprised of virtual machines, which are not usually physically attached to persistent storage. This enables those machines
to migrate across hosts, backed by network attached drives. We can consider this model for optimizing real-world distributed consensus.

Roughly this suggestion is described by Lamport himself in [paxos made simple]. It is applied in practice in [FoundationDB]. I will discuss how it applies to Raft and
systems derived from Raft. The raft paper suggests monolithic nodes: with responsibility for electing a leader among themselves, persisting the log, and applying log
entries to their state machine. This is roughly what we see in practice for many implementations of raft. But this means that servers must be optimized for high-performance
both in running the state machine as well as keeping the consensus log. These are, at times, conflicting goals.

For example, consider the case where we seek to update the state machine implementation. The actual log storage and replication part of the implementation should not have to change.
We can avoid some downtime if only the state machine is swapped. In addition, we may gain an advantage from being able able to use cpu-optimized hardware for the state machine servers,
and storage-optimized hardware for the log structure servers.

[Disk paxos], which is used within FoundationDB, shows that Paxos can be adapted to work with seperate processors and network-attached-disks. Processors are responsible for
driving the consensus process as well as in-practice run the state machine, while disks take the responsibility for storage. Similar to regular Paxos, Disk paxos needs a single
processor to be elected as a "leader" in order to make progress, but still guarantees safety in the presence of crash-stop failures. Unlike regular Paxos, Disk paxos can tolerante
failures of any number of processors as long as enough disks are available.

We can consider adapt a similar model for Raft, placing the storage responsibility for logs upon "log storage" servers, or scribes. We will seperate out two responsibilities/classes of server:
workers and scribes. Workers maintain state machines and can become leader, but do not persist logs or participate in quorums or voting. Scribes persist logs and participate in quoroms. This
is similar to [paxos made simple] and many implementations of Paxos.

We need to modify the Raft leader election. When trying to become elected as a leader, all workers may be arbitrarily far behind. Workers must be able to
fast forward when necessary. In the response to RequestVote, scribes respond with the `(index, term)` of the tail of their log. A candidate worker that sees its log is not far enough ahead to
win an election can then pick a scribe to read recent log entries from.

In the state where a leader is elected, this looks like Raft but with a distinct leaders (workers) and followers (scribes). Other workers act as learners, following the leader but not participating in consensus votes.

In the state where a leader is not elected, learners eventually convert to candidate state and request votes from scribes. They learn if their log is too far behind, and then are individually responsible for catching up
before restarting their election.

## contextualizing this proposal

While distinct roles for participants is common in the discussion of Paxos, it is not considered much for Raft.

In ["Paxos vs Raft: Have we reached consensus on distributed consensus?"][paxos fried raft], Howard and Mortier show many similarities between Raft and Paxos.
In my opinion, many of the advantages of Raft come from its simplified presentation and one key optimization to leader election (log transmission is not necessary).

We bring one key optimization from Paxos into the Raft world, and find many of the advantages of each. The leader election mechanism is similar to Raft in that
log transmission is not part of the critical path for leader election. Unlike Raft, log transmission is sometimes required during the election process, but happens
"out of band" from the election, is not safety critical, and occurs only as-needed.

[raft]: https://raft.github.io/raft.pdf
[paxos]: http://lamport.azurewebsites.net/pubs/lamport-paxos.pdf
[Disk paxos]: https://lamport.azurewebsites.net/pubs/disk-paxos.pdf
[paxos made simple]: https://lamport.azurewebsites.net/pubs/paxos-simple.pdf
[FoundationDB]: https://cacm.acm.org/magazines/2023/6/273229-foundationdb-a-distributed-key-value-store/fulltext
[paxos fried raft]: https://dl.acm.org/doi/pdf/10.1145/3380787.3393681
