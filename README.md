# af: anti-fragile systems toolkit

collection of ideas for building anti-fragile systems (distributed systems that tolerate failures and self-heal).

ideally, we can get to a place where we can replace [remote key-value stores](https://pages.cs.wisc.edu/~rgrandl/papers/link.pdf)
and our reliance on the 12-factor app. would be cool if we can keep application state in-memory without loosing horizontal scalability
or drastically increasing complexity of application development. 

this is, of course, not straightforward to do! but hopefully by creating strong components for distributed systems, we can get there.


things that we probably need:

- gossip-based membership detection
- network coordinates
- scalable consensus mechansim, with support for rolling updates of nodes
- first-class support for [network simulation](https://sled.rs/simulation.html)
