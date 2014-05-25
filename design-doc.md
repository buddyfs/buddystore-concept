# **BuddyStore**

Anup Chenthamarakshan, Narendran Thangarajan, Vidya Kirupanidhi

## Peer-to-peer Key-Value Store Design

The primary challenge in implementing a KV store in a P2P environment is that the protocol should handle arbitrary nodes joining and leaving the network. We considered classic P2P routing protocols like Kademlia, Pastry and Chord as potential lookup protocols. We chose Chord due to our familiarity with the protocol and implementation, however the proposed idea should work seamlessly over Kademlia and Pastry as well. BuddyStore is implemented as a distributed key-value store, by means of a DHT, layered on top of Chord lookup protocol. All the peers in the network will be part of a global Chord ring as found in the Chord File System [1]

In our current design, each writer has a designated "write ring" for itself (this ring is identified by the public key of the writer). So, there is a global ring with K overlapping sub-rings where K is the number of users in the system. Fig. 1 shows a ring configuration with three users in the system. Each ring corresponds to one writer (highlighted) and it contains only the nodes that have agreed to store the keys written by the writer. When the writer goes down, the remaining nodes in the corresponding ring make sure that there are f+1 replicas of all the keys at any point in time.

![alt tag](https://raw2.github.com/narendran/buddystore-concept/master/images/virtual-rings.png)
Fig. 1 : Multiple Virtual Rings

The global ring serves 2 purposes - looking up nodes on the network (using the default Chord lookup mechanism) and looking up writer rings (by using a lightweight DHT on top of Chord).

This design scales well since each node has to send at most O(K) messages as part of the Chord ring maintenance protocol (stabilize, check_predecessor), where K is the number of write rings that the node is part of. However, this design doesn’t have strong consistency properties. One scenario where the transparency breaks is when a client fails while writing. Different set of successors for that key K will have different values for the same key K which breaks consistency. So multiple reads for the same key may return different values based on which replica is chosen during the read.

Our current design uses the concept of distributed lease managers to implement atomic updates using write leases. Every sub-ring on the network has one or more logical lease managers each of which is responsible for a distinct subset of the keyspace contained in the ring. As shown in Fig. 2, a logical lease manager is a set of nodes comprising of a primary and multiple backups using the primary copy method. Logical lease managers are at well known locations on the sub-ring. Every sub-ring starts with a single lease manager and this lease manager can split the keyspace it is handling (for load balancing) by setting up additional lease managers and delegating parts of its keyspace to them. This process can be performed recursively as well.

![alt tag](https://raw2.github.com/narendran/buddystore-concept/master/images/distributed-lm.png)
Fig. 2 : Distributed Lock Manager

Having a lease manager allows updates to be performed on keys with required sequential consistency characteristics. Multiple simultaneous write operations on the same key are prevented by using a write lease on the key. Cases where clients holding write leases fail are handled effectively by setting very short write lease durations (on the order of minutes). In order to achieve sequential consistency, we also associate a per-key monotonically increasing version number to the value stored in every set operation.

## Operations :

* CREATE : When a client issues a Set(key, value) 
   * The Client C hashes its public key
   * C looks up LockManager LM of the ring using the hashed key
   * C requests WriteLease(key)
   * LM verifies that there are no outstanding write leases for the same key in its lease table.
   * If there is an outstanding lease that conflicts with the current request, respond with E_LOCKED. Client will retry the operation after a random period of time (with backoff).
   * Else, return (next_writable_version)
   * C looks up for the data replicas where the key is stored using hash(key)
   * C broadcasts a Write(key, next_writable_version, value) to each of the replicas and waits for an acknowledgement from all of them.
   * If C receives all ACKs
      * C sends CommitRelease(key, next_writable_version) to LM
      * LM commits the version change in its table and sends an ACK to C
      * All future Read requests for key at the LM will be responded to with next_writable_version
      * C returns to calling process with a success code
   * If C does not receive all ACKs within a timeout
      * C sends AbortRelease(key, next_writable_version) to LM C retries the write operation
      * Next WriteLease to the LM will be responded to with (next_writable_version+1)

* READ : When a client issues a Get(key) the following sequence of operations ensue.
   * The Client C hashes its public key 
   * C looks up for the Lock Manager LM of the ring.
   * C sends a Read(key) request to LM.
   * LM returns the committed version of the key requested.
   * C looks up for the node based on the hash(key).
   * C sends a Read(key,version) to the resulting node N.
   * N returns the value for the corresponding (key,version).

* JOIN : When an arbitrary node joins the system, the following operations ensue.
   * According to Chord protocol, the new node looks up the Chord ring for its own node ID.
   * For each public key in its constraint list, it looks up the global ring to get the list of nodes in the corresponding public key’s virtual ring.
   * The new node joins all the virtual rings.
   * If the new node discovers that it is the Lock Manager for that virtual ring, then it takes over as the new Lock Manager using a slightly extended protocol.

* LEAVE : When a node leaves the system, the operations that ensue are as followed in the Chord protocol.

### Software Artifacts

* Distributed KV store
   * Uses the implementation of Chord lookup protocol from go-chord [2]

* Filesystem
   * Uses a FUSE library implemented in Golang [3]


### Targeted goals

* Sequential Consistency
    * Any get operation on a key X returns the value stored by the most recent set operation on the key X

* Scalability
    * Performance of operations on the key-value store will not appreciably deteriorate with the total number of nodes or keys in the system.
    * If an individual node is part of more than an empirical threshold number of rings, then the scalability guarantees might be violated. We will determine this empirical threshold during the testing phase.

* Availability
    * Replicated copies of every key-value entry will be maintained in multiple nodes.

* Fault Tolerance
    * Transparent migration of data to new replicas will keep the system operational in the face of failures.

* Support for constraint
    * User will be able to control whose data it stores.

## References :

[1] Wide Area Cooperative Storage with CFS - SOSP ‘01, Dabek et al.

[2] [https://github.com/armon/go-chord](https://github.com/armon/go-chord)

[3] [https://github.com/bazillion/fuse](https://github.com/bazillion/fuse)
