# Distributed Systems: Theory & Practice

## Why do we have distributed systems in the first place (5 mins)

- Case 1: you're starting a startup, at first it's just a Node-JS server hosted on your own personal laptop. As you gain more user, you find that your computer keeps running out of RAM and the website keeps going down. So you buy a larger computer with 60GB RAM (this is called vertical scaling), but eventually you find that even though you have enough RAM, your CPU is not fast enough to process all the requests and your disk/network IO speed is bottlenecking the efficiency of your web server. So you decided to buy more computers and distribute the workload to each machine (this is called horizontal scaling). But one day, your apartment had a power outage, all of your machines go down at once, and when the power is back on, you find out that all of your user's data have been lost. Your investors are extremely angry at you. So you decide to sell your car and rent out even more machines in multiple regions to make backups of your user's data (this is called replication). Finally, your startup becomes successful and begins to expand internationally, but the two way traffics between Asia and US are too slow, so you rent out machines in Asia to create a mid-layer cache for all traffics coming from Asia. 

- Case 2: you're a research scientist at NCSA working on genomic research. You need to train a machine learning model on billions of DNA sequences to find correlations between mutations and diseases. The dataset you have is too large to sit on one machine, so you put them on a cluster of computers using a distributed filesystem like HDFS. But because your data are on different machines, you need to find a way to perform computation on your distributed dataset, and after some digging, you realize that the computation you need is embarrassingly parallel, so you adopt a MapReduce system like Spark to distribute computation. 

### What distributed system does:

1. Level of abstraction
2. Manage Scale: horizontal scaling vs vertical scaling (most websites with large traffics choose horizontal scaling over vertical scaling)
2. Manage Fault: backups
3. Physical Distance (shared resource, ex: Internet, Google Docs, ...)
4. Distributed Computation ("BIG DATA")
5. It's really a study of uncertainty (things fall apart)

### Best example of distributed systems:

- "Drunk friends trying to make plans via text message." (Kyle Kingsbury)

## Distributed system concepts to understand (for a happier and fuller life) (20 mins)

- In a distributed system:
	- Things happen fast within a node
	- Things happen slow (if at all) between nodes
	- Nodes can fail (either crash or Byzantine)
	- Network is almost always unreliable

### Asynchronous Network

- Network in real world is asynchronous
	- Unbounded message delays
	- No global clocks
		- Why is there no central clock?
		- Well, first, what is time?
		- Time in physics is relative.
		- Time in distributed system is also relative.
		- Problems with using wall clock:
			- Clock skew (each machine has different time).
			- Clock drift (hardware ticks at different rate).
			- Even synchronization (NTP) does not work very well when there is network delay and network loss (delta > 128ms).
			- Even with accurate wall clock, threads can sleep, processes can sleep, OS can sleep, so just don't.
		- So what do we use?
			- Logical ordering
			- Logical timestamp: vector clock
	- IP network is asynchronous, but in practice, we set a bound to message delay and say "after 30s, we don't wait for message anymore" to offset some issues
	- Protocols:
		- TCP prevents duplications and order packages for you in the context of a single TCP connection.
		- But you'll probably going to need more than one TCP connections, and more than one machine.
		- So most of the times, you'll have to reorder messages by encoding your own sequence vector on top of TCP.
		- UDP is just not used in general. Package will be arbitrarily dropped and duplicated, and security and debugging is hard. (Netflix uses TCP).
		- UDP is used only when TCP overhead is absolutely prohibitive.

### Failures

- Lesson zero in distributed systems and cloud computing: failure happens all the time.
	- "AWS -- Chaos Monkey as Service" from talk by Netflix, talking about how AWS kills more micro-services run by Netflix than Netflix.
	- How many have used AWS: ...instances can disappear, zones can fail, regions can become unstable, network is lossy, requests can bounce between regions...
- The main difference between thinking about distributed systems & single-machine systems: failure happens often (especially partial failure)
	- "Distributed systems aren't just characterized by latency, but by recurrent, partial failure"
	- What makes distributed systems hard is failure, not asynchrony
	- When implementing something distributed, a naive engineer may say things like "just write to all the machines" or "just keep retrying the write command until it succeeds". But the problem is that networked system could fail partially. What if one of the writes succeeded while others failed, what if the same write occurs twice due to network delay, what if garbage collection pauses and make one of the node "disappear", what if a slow disk drive on one machine causes the communication protocol in the whole system to slow down. These partial failures are hard to reason about, and it's much much simpler to deal with problems on a single machine.
- Goal of distributed systems: tolerate partial failure
	- Availability of a system
		- 99.9% availability = 8.76 hour/year
		- 99.99% availability = 52.6 mins/year
		- 99.999% availability = 5.26 mins/year
	- FLP Impossibility (worst case)

### Consistency
- Data replication
	- for availability (ex: local cache)
	- for reliability (ex: backup database)
	- for scalability (ex: load-balancing)
- Price to be paid: any time we have replication, we need to think about consistency (so in reality, we almost always need to think about consistency)
	- Def: all nodes (partitions) return the same and latest data
	- Why is this important?
		- Depends on application: in good cases users get staled data, in bad cases get corrupt memory and business logic nightmares
			- Ex: FB/Uber/transaction
	- Why is this hard to do?
		- Latency: message ordering (did write(a) happen first or write(b)), remember: no global clock
		- Failure: message lost (write(a) sent to one node but not the other), machine fails and recovers, etc
	- How do we measure it? 
		- Consistency models (like guarantees)
			- In order from weak to strong guarantees:
			- Eventual Consistency
				- Each read will return some historical write, but eventually all nodes will all have the same values.
			- Causal Consistency
				- If A->B, then A must be finished before B
				- Operations can be linked by a DAG
			- Sequential Consistency
				- Operations of all processors happen are executed in *some* sequential order (same thing as serializable database transactions)
					- we don't care the order, just *some* order
			- Linearizability (the distributed holy grail)
				- All operations happen as if instantaneously (at the linearization point), and it preserves real-time order (if a finishes before b starts, then a must happen before b)
			- Very important to understand consistency model (less so for how to achieve them), because you see them everywhere in application (Ex: choosing database)

### Trade-Offs (at service level, given a capacity):
- Availability
- Consistency (Accuracy)
- Partion-Tolerance (Latency)

### CPA Theorem

- Paper Link: https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf

- Brewer's original conjecture: suppose for a web server
	- Consistency: each server returns the right response
	- Availability: each request eventually receives a response
	- Partition-Tolerance: when communication is faulty (message delay/lost), system still functions

- Consensus: the heart of replicated state machine approach
- FLP Impossibility: it is impossible to achieve fault-tolerant agreement.
	- Agreement
	- Validity
	- Termination
- Sacrifice either availability or consistency or both
	- Best Effort Availability
		- Good approach if the underlying communication network is reliable most of the time (i.e. in a data center)
		- Chubby (Google): using Paxos, continues to work if no more than half the servers fail
	- Best Effort Consistency
		- When users must have immediate response in all situations.
		- Ex: websites (think cached image/video/posts/etc where new contents take time to propagate but it's okay)
	- Tradeoff:
		- Pick consistency: payment system, reservation system, trading system, etc
		- Pick availability: content sites, streaming system
	- Caveat: different part of a system has different requirements
		- Data partitioning
			- search bar: want high availability, okay to be inconsistent in some anomalous cases
			- product information: user will tolerate some out-of-date inventory information
			- checkout-payment-shipment: must be strongly consistent
		- Operation partitioning
			- read/write: want high availability on read, want high consistency on write
		- User Partitioning
			- partition users by slow/fast network connections (ex: Netflix)
		- Hierarchical Partitioning, etc...
		- Lesson: software architecture should explore different consistency/availability tradeoffs at different sub-components to come up with the solution that best meets the business need.
- CAP Theorem Today
	- New challenges: 
		- Scalability
			- there seems to be a triangular trade-off between scalability, consistency, and availability
			- ex: this is why, even within data centers (think Google cloud/AWS), it's still hard to efficiently scale strongly consistent protocols!
		- Attacks
			- CAP worries about network partitions (aka when some servers can't communicate reliably), but now we're seeing server network attacks like DOS, DDOS, and disrupted major services (like DNS). So it's even harder to ensure safety/liveliness.

---

## How real people in real world design real distributed systems (25 mins)

### Case 1: Trouble with Sharding

- Evolution of database architecture
	- Single-Node Storage
	- Read Replications
		- Now we need to start worrying about consistency (tradeoff between latency and consistency)
	- Sharding with Replications
		- Split database into partitions (shards)
			- Each shard has its own read replications
			- Ex: by primary key, by region (why is this bad?), by catalog, etc
- Something wicked this way comes:
	- Problems emerge
		- How to partition evenly?
			- Hard to estimate load on each shard
			- Hot keys
		- Keys change over time, need to re-balance (aka: remapping)
			- And how do client application know which shard to look for a key? (since each shard knows nothing about the other shards, application layer is responsible for fetching data)
		- Traffic change over time, need to add or remove shards
			- Finite memory/network bandwidth
				- Node exhaustion
		- And how about failures? If one shard goes down, we need to immediately re-shard!
- Case Study: Foursquare
	- In 2010, Foursquare experienced a 17 hours downtime caused by trouble with database sharding.
	- At the time, Foursquare had about 3 million users and 18,000 checkins per day.
	- Foursuqare uses MongoDB to store user data on two EC2 nodes each with 66GB of RAM.
	- The partition between two machines was not even (more record went into the first shard than the second). Shard on first node grew larger than RAM, so the OS began paging to disks, but page swaps are slow, causing the backlog to queue up very fast and bringing the site down.
	- To resolve the issue, engineers added a third shard, but this did not help. Because MongoDB uses 4K virtual memory page (as Kernel does as well), and each checkin is only about 300 bytes, so the 4K memory is only freed when all data on the page is released. The result is that no memory was actually freed.
	- Usually MongoDB has a compaction feature (which defragment and rewrite data on disk), but EC2's network disk storage is slow, compaction was way too slow.
	- Eventually, the issue was resolved by reloading the system from backups and resharding across three shards.
- Lessons learned:
	- The real problem is Redis's low level cache design, but... we learned that
		- Partial failure can bring down the whole system, traffic is hard to estimate, re-sharding is unreliable in emergencies, recovery can be slow, backup is critical.
- An alternative solution for database partition: Hash Rings (Consistent Hashing)
	- Think of it as a distributed implementation of a hash table
	- Goal: 
		- Sometimes nodes die, sometimes node get added, so hash changes (ex: with mod-N hashing, 1003%10 = 3 but 1003%9 = 4), mapping becomes inconsistent unless we perform remapping constantly
		- When adding or removing a server, we only want to move 1/nth keys and not move keys that don't need to be moved (ex: moving keys between old servers should be unnecessary)
	- Solution:
		- Hash both the key and the node
		- Hash each node into an integer (a point on our ring), when a key needs to be looked up, hash the key into an integer (also a point on our ring), then just go around the circle and find the node with the smallest hash greater than the key's hash.
		- Ex: node hash to 0, 80, 177, if hash(a)=13 then key a sits on node 80
		- To add a node, say node 40, we only need to move the keys 1-40 from node 80 (no need to re-distribute any other keys on other nodes)
		- To remove a node, again, we just need to move the keys off the node and move them to the node with the smallest number greater than the node to remove.
		- Potential problem: 
			- Clustering (some nodes will store disproportionately large number of keys): solve it by adding each server to the ring more than one times at different places
			- Fault-tolerance: store keys on multiple locations (replication)
		- Trade-off: time complexity
			- Hash table vs Consistent hashing
			- Adding/removing a key O(1) vs O(log(N))

### Case 2: Trouble with Locking:

- Distributed Locking
	- In a single-machine system, suppose a mutex unlock call fails with error -1, then the process is unstable and we can just crash it.
	- But in a distributed systems, if you want to implement a mutex lock, dealing with failure must be built in into the protocol.
		- Approach 1: broadcast to everyone, and ask for a token (message can get lost, node can fail, also inefficient, congestion, timeout... very messy)
		- Approach 2: central lock manager that manages token, send it to a node as permission to enter and wait to get it back (problem: message can get lost, node can fail; ex: a node receives the token then dies... still messy)
		- Approach 3: central lock manager still manages token, but each token has a lease, after the lease ends the lock is released (btw, this is how Redis does locking, which is why Redis is not good for locking) (problem: pause-the-world = the client which acquires the lock may be paused for an extended period of time without knowing, due to garbage-collector (even without GC this can still happen if for instance your process tries to read a page not yet allocated and triggers a page fault, and the kernel pauses your world until the page is loaded))
		- Approach 4: same as approach 3 but with fencing, so attack a unique number (counter) to each token and let storage service actively check the token to make sure any action performed is not expired (problem: so you think incrementing a counter is easy, hmmm? for fault-tolerance you want to keep the counter on more than one node, which means you'll need a consensus algorithm just to correctly increment counter)
		- Approach 5: Paxos
			- Industry-wide gold-standard for consensus algorithm
			- Problem: hard to understand and very hard to implement (so much so I'll skip it in lecture but look it up by yourself if you're interested)
			- Existing implementation: Apache Zookeeper

#### Practical Tips (Food for Thoughts)

1. Do not distribute
	1. Local system have very reliable primitives (lock, thread, etc), use them
	3. Go distributed only when the service cannot tolerate a single node's guarantees
2. Use existing distributed systems
	1. Don't roll your own system
	2. Use Cassandra, Zookeeper, Kafka, Redis, etc...
	3. But understand them first
3. Accept failure
	1. Accept the fact that even large companies with the best distributed systems can't avoid failures
	2. Be ready to put out fire, recover above the system level, be ready to call the customer and apologize
	3. Always always always have backup. This allows you to recover in a matter of minutes to days.
4. Design for specific application
	1. Understand your problem
		1. What consistency do you want? (Ex: linearizability for user OP, but eventual consistency for social feed)
		2. What availability do you want?
		3. Is latency an important issue?
	2. Break down your application into components to make trade-offs
	3. Estimate your capacity, collect metrics, think of it as a game of resource allocation
