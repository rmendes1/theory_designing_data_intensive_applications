# Replication

Replication involves maintaining copies of data across multiple machines to reduce latency, increase availability, and improve read throughput. This chapter focuses on datasets small enough to fit on a single machine, with partitioning for larger datasets discussed later.

**Challenges in Replication**:

The primary difficulty lies in handling changes to replicated data. Three algorithms are examined: single-leader, multi-leader, and leaderless replication, each with trade-offs. Key considerations include synchronous vs. asynchronous replication and handling failed replicas.

**Historical Context and Misunderstandings**:

Replication principles, rooted in 1970s research, remain relevant. Despite recent mainstream adoption of distributed databases, misconceptions around concepts like eventual consistency persist. Guarantees such as read-your-writes and monotonic reads are also discussed.

## **Leaders and Followers in Replication**:

- **Replica Roles**:
    1. **Leader (Master/Primary)**:
        - Handles all write requests from clients.
        - Writes new data to its local storage.
        - Sends data changes to followers via a replication log or change stream.
    2. **Followers (Read Replicas/Slaves/Secondaries)**:
        - Receive and apply changes from the leader in the same order.
        - Maintain a local copy of the database.
        - Serve read requests but do not accept writes.
- **Client Interaction**:
    - **Writes**: Only accepted by the leader.
    - **Reads**: Can be served by either the leader or any follower.

**Applications of Leader-Based Replication**:

- **Databases**:
    1. **Relational Databases**:
        - PostgreSQL (since version 9.0).
        - MySQL.
        - Oracle Data Guard.
        - SQL Server’s AlwaysOn Availability Groups.
    2. **Nonrelational Databases**:
        - MongoDB.
        - RethinkDB.
        - Espresso.
- **Other Systems**:
    1. **Distributed Message Brokers**:
        - Kafka.
        - RabbitMQ (highly available queues).
    2. **Network Filesystems and Replicated Block Devices**:
        - DRBD.

![image.png](attachment:1da9f2ae-fba0-4cad-bc0f-763d743c75e6:image.png)

### **Synchronous Versus Asynchronous Replication**:

- Replication can be **synchronous** or **asynchronous**, depending on whether the leader waits for followers to confirm writes before acknowledging success to the client.
- In **synchronous replication**:
    1. The leader waits for a follower to confirm receipt of the write.
    2. Ensures the follower has an up-to-date copy of the data.
    3. If the synchronous follower fails or is slow, writes are blocked until it recovers.
- In **asynchronous replication**:
    1. The leader sends writes to followers but does not wait for confirmation.
    2. Followers may lag behind the leader, especially under high load, network issues, or failure recovery.
    3. Writes may be lost if the leader fails before replication completes.
- **Trade-offs**:
    1. **Synchronous Replication**:
        - **Advantages**: Guarantees data consistency and durability.
        - **Disadvantages**: Vulnerable to system halts if the synchronous follower fails.
    2. **Asynchronous Replication**:
        - **Advantages**: Higher performance and availability, as the leader can continue processing writes even if followers lag.
        - **Disadvantages**: Risk of data loss if the leader fails before replication completes.
- **Practical Configurations**:
    - Often, only **one follower is synchronous** (semi-synchronous), while others are asynchronous.
    - If the synchronous follower fails, an asynchronous follower is promoted to synchronous to maintain data durability.
- **Use Cases**:
    - Asynchronous replication is widely used, especially with many followers or geographically distributed systems, despite the risk of data loss.

**Research on Replication**:

- **Challenges**: Asynchronous replication risks data loss if the leader fails.
- **Innovations**:
    1. **Chain Replication**: A synchronous replication variant used in systems like Microsoft Azure Storage.
    2. **Consistency and Consensus**: Strong theoretical connections exist between replication consistency and consensus algorithms, explored further in later chapters.

![image.png](attachment:f902948c-5e23-43cf-ae92-6344096f2d67:image.png)

### **Setting Up New Followers**:

- **Objective**:
    - Add new followers to increase replicas or replace failed nodes while ensuring an accurate copy of the leader’s data.
- **Challenges**:
    - Direct file copying is insufficient due to ongoing writes by users, which can result in inconsistent data (since there would be different copies from the same file).
    - Locking the database for consistency conflicts with high availability goals.
- **Process**:
    1. **Take a Consistent Snapshot**:
        - Capture a snapshot of the leader’s database at a specific point in time, ideally without locking the entire database.
        - Tools like `innobackupex` for MySQL may be required.
    2. **Copy the Snapshot**:
        - Transfer the snapshot to the new follower node.
    3. **Request Data Changes**:
        - The follower connects to the leader and requests all changes made since the snapshot.
        - The snapshot must be associated with a precise position in the leader’s replication log (e.g., PostgreSQL’s log sequence number or MySQL’s binlog coordinates).
    4. **Catch Up**:
        - The follower processes the backlog of changes until it is up-to-date.
        - It then continues to handle real-time changes from the leader.
- **Implementation Variability**:
    - The process can range from fully automated to manual, multi-step workflows, depending on the database system.

### **Handling Node Outages**:

- **Objective**:
    - Maintain system availability during node failures, whether unexpected (e.g., crashes) or planned (e.g., maintenance).
- **Follower Failure - Catch-up Recovery**:
    1. Followers keep a log of data changes received from the leader.
    2. On recovery, the follower uses its log to identify the last processed transaction.
    3. It requests all changes made during the outage from the leader and applies them to catch up.
- **Leader Failure - Failover**:
    - **Process**:
        1. **Detect Leader Failure**:
            - Use timeouts to determine if the leader is unresponsive.
        2. **Choose a New Leader**:
            - Elect the replica with the most up-to-date data changes.
            - This is a consensus problem (discussed in Chapter 9).
        3. **Reconfigure the System**:
            - Redirect client writes to the new leader.
            - Ensure the old leader becomes a follower if it rejoins.
    - **Challenges**:
        1. **Data Loss in Asynchronous Replication**:
            - Unreplicated writes from the old leader may be discarded, violating durability expectations.
        2. **Inconsistent State Across Systems**:
            - Example: GitHub incident where an out-of-date follower reused primary keys, causing inconsistencies between MySQL and Redis.
        3. **Split Brain**:
            - Two nodes may believe they are the leader, leading to data loss or corruption if both accept writes.
        4. **Timeout Configuration**:
            - Long timeouts delay recovery; short timeouts risk unnecessary failovers due to temporary issues like network glitches or high load.
- **Failover Trade-offs**:
    - Automatic failover is complex and error-prone, leading some teams to prefer manual failover.
- **Broader Implications**:
    - Node failures, unreliable networks, and trade-offs around consistency, durability, availability, and latency are fundamental challenges in distributed systems, explored further in Chapters 8 and 9.

### **Implementation of Replication Logs**:

- **Overview**:
    - Leader-based replication can be implemented using different methods, each with its own trade-offs.

#### **1. Statement-Based Replication**:

- **Mechanism**:
    - The leader logs every write request (e.g., `INSERT`, `UPDATE`, `DELETE`) and forwards the SQL statements to followers.
    - Followers execute these statements as if received from a client.
- **Issues**:
    1. **Nondeterministic Functions**:
        - Functions like `NOW()` or `RAND()` produce different values on each replica.
    2. **Order Dependency**:
        - Statements relying on autoincrementing columns or existing data must execute in the same order on all replicas.
    3. **Side Effects**:
        - Triggers, stored procedures, or user-defined functions may behave differently unless deterministic.
- **Workarounds**:
    - Replace nondeterministic functions with fixed values during logging.
    - Used in MySQL before version 5.1; now defaults to row-based replication for nondeterministic statements.

#### **2. Write-Ahead Log (WAL) Shipping**:

- **Mechanism**:
    - The leader appends writes to a log (used for crash recovery) and ships this log to followers.
    - Followers rebuild the same data structures as the leader using the log.
- **Applications**:
    - Used in PostgreSQL, Oracle, and other databases.
- **Disadvantages**:
    1. **Low-Level Details**:
        - The log describes disk-level changes, tightly coupling replication to the storage engine.
    2. **Version Compatibility**:
        - Leader and followers must run the same database version, complicating zero-downtime upgrades.

#### **3. Logical (Row-Based) Log Replication**:

- **Mechanism**:
    - Uses a logical log decoupled from storage engine internals.
    - Logs row-level changes:
        - **Inserts**: New values of all columns.
        - **Deletes**: Information to uniquely identify the row (e.g., primary key).
        - **Updates**: Row identifier and new column values.
- **Advantages**:
    1. **Version Independence**:
        - Allows leader and followers to run different database versions or storage engines.
    2. **External Use**:
        - Easier for external systems (e.g., data warehouses) to parse, enabling change data capture.
- **Applications**:
    - MySQL’s binlog (in row-based replication mode).

#### **4. Trigger-Based Replication**:

- **Mechanism**:
    - Uses database triggers to execute custom application code on data changes.
    - Changes are logged into a separate table and replicated by an external process.
- **Use Cases**:
    - Replicating subsets of data, replicating across different databases, or implementing custom conflict resolution.
- **Disadvantages**:
    1. **Higher Overhead**:
        - Greater performance impact compared to built-in replication.
    2. **Complexity**:
        - More prone to bugs and limitations.
- **Applications**:
    - Tools like Oracle GoldenGate, Databus for Oracle, and Bucardo for Postgres.

#### **Summary**:

- **Statement-Based**: Simple but problematic for nondeterministic operations.
- **WAL Shipping**: Efficient but tightly coupled to storage engine.
- **Logical Logs**: Flexible and decoupled, enabling version independence and external use.
- **Trigger-Based**: Highly flexible but complex and resource-intensive.

## **Problems with Replication Lag**:

- **Context**:
    - Replication is used for fault tolerance, scalability, and reduced latency.
    - In leader-based replication, writes go to the leader, while reads can be distributed across followers.
    - Asynchronous replication is common for scalability but introduces replication lag, leading to temporary inconsistencies (eventual consistency).

### **1. Reading Your Own Writes (Read-After-Write Consistency)**:

- **Problem**:
    - Users may not see their own writes immediately if they read from a lagging follower.
    - Example: A user submits data but sees outdated information when reading from a follower.
- **Solutions**:
    1. **Read from Leader for User-Specific Data**:
        - Always read the user’s own data (e.g., profile) from the leader.
    2. **Time-Based Routing**:
        - Route reads to the leader for a short period after a write (e.g., one minute).
    3. **Track Write Timestamps**:
        - Clients remember the timestamp of their last write and ensure replicas are up-to-date before reading.
    4. **Cross-Device Consistency**:
        - Centralize metadata to track updates across devices and route requests to the same datacenter.

### **2. Monotonic Reads**:

- **Problem**:
    - Users may see data moving "backward in time" if they read from different replicas with varying lag.
    - Example: A user sees a comment appear and then disappear on subsequent reads.
- **Solution**:
    - Ensure each user always reads from the same replica (e.g., by hashing user IDs to select replicas).

### **3. Consistent Prefix Reads**:

- **Problem**:
    - Causality violations occur when writes are observed out of order due to replication lag.
    - Example: A listener hears an answer before the question is asked.
- **Solution**:
    - Ensure causally related writes are read in the correct order.
    - Techniques:
        1. Write causally related data to the same partition.
        2. Use algorithms to track causal dependencies (e.g., "happens-before" relationships).

### **Handling Replication Lag**:

- **Application-Level Solutions**:
    - Implement stronger guarantees (e.g., read-after-write) in application code, though this adds complexity.
- **Database-Level Solutions**:
    - Use transactions to provide stronger consistency guarantees.
    - While single-node transactions are well-established, distributed databases often sacrifice transactions for scalability, leading to eventual consistency.
- **Trade-offs**:
    - Replication lag is inevitable in distributed systems, but its impact can be mitigated through careful design.
    - Transactions and causal consistency mechanisms can help balance performance and correctness.

## **Multi-Leader Replication**:

- **Overview**:
    - Extends leader-based replication by allowing multiple nodes (leaders) to accept writes.
    - Each leader forwards writes to other leaders, acting as both a leader and a follower.
    
    ![image.png](attachment:91e02b08-4eb0-4735-be4d-eafe4b1cea30:image.png)

### **Use Cases for Multi-Leader Replication**:

#### **1. Multi-Datacenter Operation**:

- **Setup**:
    - Each datacenter has its own leader, with asynchronous replication between datacenters.
- **Advantages**:
    1. **Performance**:
        - Writes are processed locally, reducing latency compared to single-leader setups.
    2. **Tolerance of Datacenter Outages**:
        - Each datacenter operates independently; replication resumes after outages.
    3. **Tolerance of Network Problems**:
        - Asynchronous replication handles temporary network interruptions better than synchronous replication.
- **Challenges**:
    - Write conflicts can occur when the same data is modified in different datacenters, requiring conflict resolution.
    - Subtle configuration issues (e.g., autoincrementing keys, triggers) can complicate implementation.

#### **2. Clients with Offline Operation**:

- **Setup**:
    - Each device (e.g., mobile phone, laptop) acts as a leader with a local database.
    - Changes are asynchronously synced across devices when online.
- **Use Case**:
    - Applications like calendar apps that need to function offline.
- **Challenges**:
    - High replication lag (hours or days) due to unreliable network connections.
    - Tools like CouchDB are designed for this mode of operation.

#### **3. Collaborative Editing**:

- **Setup**:
    - Multiple users edit a document simultaneously, with changes applied locally and asynchronously replicated.
- **Use Case**:
    - Real-time collaborative tools like Google Docs or Etherpad.
- **Challenges**:
    - Conflict resolution is required if users edit the same data concurrently.
    - Avoiding locks for faster collaboration introduces multi-leader replication challenges.

#### **Challenges of Multi-Leader Replication**:

1. **Write Conflicts**:
    - Concurrent modifications to the same data in different leaders require conflict resolution.
2. **Configuration Complexity**:
    - Subtle pitfalls with features like autoincrementing keys, triggers, and integrity constraints.
3. **Replication Lag**:
    - Asynchronous replication can lead to significant lag, especially in offline or collaborative scenarios.

## **Handling Write Conflicts in Multi-Leader Replication**:

- **Problem**:
    - Write conflicts occur when the same data is modified concurrently on different leaders, requiring conflict resolution.
    - Example: Two users editing the same wiki page title simultaneously, resulting in conflicting updates.
    
    ![image.png](attachment:18f1c5e6-cc9f-439c-9b0f-898fe6f40847:image.png)
    
### **Conflict Detection**:

- **Single-Leader vs. Multi-Leader**:
    - In single-leader systems, conflicts are detected synchronously (e.g., blocking or aborting conflicting writes).
    - In multi-leader systems, conflicts are detected asynchronously, often too late for user intervention.
- **Synchronous Conflict Detection**:
    - Not practical in multi-leader setups, as it negates the independence of writes across replicas.

### **Conflict Avoidance**:

- **Strategy**:
    - Ensure all writes for a specific record go through the same leader.
    - Example: Route a user’s requests to a designated "home" datacenter.
- **Limitations**:
    - Breaks down during leader rerouting (e.g., datacenter failure or user relocation).

### **Converging Toward Consistency**:

- **Challenge**:
    - Multi-leader systems lack a defined write order, leading to inconsistent states across replicas.
- **Conflict Resolution Strategies**:
    1. **Last Write Wins (LWW)**:
        - Use a unique ID (e.g., timestamp) to pick the "winning" write. Prone to data loss.
    2. **Replica Priority**:
        - Assign unique IDs to replicas; higher-priority replicas override lower-priority ones.
    3. **Value Merging**:
        - Combine conflicting values (e.g., concatenate alphabetically).
    4. **Explicit Conflict Recording**:
        - Store conflicts in a data structure and resolve later (e.g., via user prompts).

### **Custom Conflict Resolution Logic**:

- **On Write**:
    - Conflict handlers (e.g., in Bucardo) resolve conflicts immediately during replication.
    - Runs in the background without user interaction.
- **On Read**:
    - Conflicting versions are stored and returned to the application for resolution (e.g., CouchDB).
    - Applications can prompt users or resolve conflicts automatically.
- **Limitations**:
    - Conflict resolution is typically per row/document, not per transaction.

### **Automatic Conflict Resolution**:

- **Challenges**:
    - Custom resolution logic can be complex and error-prone (e.g., Amazon’s shopping cart issue).
- **Research and Techniques**:
    1. **Conflict-Free Replicated Data Types (CRDTs)**:
        - Data structures (e.g., sets, counters) that automatically resolve conflicts.
        - Used in Riak 2.0.
    2. **Mergeable Persistent Data Structures**:
        - Track history and use three-way merge functions (similar to Git).
    3. **Operational Transformation (OT)**:
        - Algorithm for collaborative editing (e.g., Google Docs, Etherpad).
- **Future Potential**:
    - Automatic conflict resolution could simplify multi-leader replication for applications.

### **Identifying Conflicts**:

- **Obvious Conflicts**:
    - Concurrent writes to the same field in the same record.
- **Subtle Conflicts**:
    - Example: Overlapping bookings in a meeting room system.
    - Requires careful detection and resolution mechanisms.

## **Multi-Leader Replication Topologies**:

- A replication topology defines the paths through which writes are propagated between nodes in a multi-leader setup.
- Multi-leader replication topologies vary in complexity and fault tolerance.
- All-to-all topologies offer the highest resilience but may face causality issues.
- Circular and star topologies are simpler but vulnerable to single points of failure.
- Proper conflict detection and causality handling (e.g., version vectors) are critical for maintaining consistency.

![image.png](attachment:8f6633f6-960d-4733-bc94-55516c841635:image.png)

### **Types of Topologies**:

#### **1. All-to-All Topology**:

- **Description**:
    - Every leader sends its writes to every other leader.
- **Advantages**:
    - High fault tolerance due to multiple communication paths.
- **Disadvantages**:
    - Potential for causality issues if writes arrive out of order.

#### **2. Circular Topology**:

- **Description**:
    - Each node forwards writes to one other node in a circular manner.
- **Advantages**:
    - Simpler than all-to-all.
- **Disadvantages**:
    - A single node failure can disrupt replication for all nodes.
    - Requires manual reconfiguration to bypass failed nodes.

#### **3. Star Topology**:

- **Description**:
    - A designated root node forwards writes to all other nodes.
- **Advantages**:
    - Centralized control over replication flow.
- **Disadvantages**:
    - The root node becomes a single point of failure.

#### **4. Tree Topology**:

- **Description**:
    - A generalization of the star topology, with hierarchical forwarding of writes.
- **Advantages**:
    - Scalable for large systems.
- **Disadvantages**:
    - Vulnerable to failures at higher levels of the hierarchy.

### **Challenges in Topologies**:

![image.png](attachment:22737738-b3b7-4d61-8c29-1fb08bc760cf:image.png)

#### **1. Replication Loops**:

- **Solution**:
    - Tag writes with unique node identifiers to prevent infinite loops.

#### **2. Fault Tolerance**:

- **Circular/Star Topologies**:
    - Vulnerable to single-node failures disrupting replication.
- **All-to-All Topologies**:
    - More resilient due to multiple paths for replication.

#### **3. Causality and Ordering**:

- **Problem**:
    - Writes may arrive out of order due to varying network speeds.
- **Example**:
    - An update may arrive before the corresponding insert, violating causality.
- **Solution**:
    - Use **version vectors** to track dependencies and ensure correct ordering.

#### **Implementation Issues**:

- **Conflict Detection**:
    - Many multi-leader systems (e.g., PostgreSQL BDR, Tungsten Replicator for MySQL) have poor support for causal ordering and conflict detection.
- **Recommendations**:
    - Carefully review documentation and test the system to ensure it meets consistency and fault-tolerance requirements.

## **Writing to the Database When a Node Is Down**:

- **Leaderless Configuration**:
    - No failover process exists.
    - Clients send writes to all replicas in parallel.
    - A write is considered successful if a quorum of replicas (e.g., 2 out of 3) acknowledges it.

### **Handling Stale Reads**:

- **Problem**:
    - When a failed node recovers, it may have missed writes and serve stale data.
- **Solution**:
    - Clients read from multiple replicas in parallel and use version numbers to determine the most recent value.
    
    ![image.png](attachment:7b91f070-c872-4bd7-87c6-9c45c7943d53:image.png)

### **Replication Repair Mechanisms**:

#### **1. Read Repair**:

- **Process**:
    - During a read, if a stale value is detected, the client writes the newer value back to the outdated replica.
- **Use Case**:
    - Effective for frequently read data.

#### **2. Anti-Entropy Process**:

- **Process**:
    - A background process continuously compares replicas and copies missing data.
- **Use Case**:
    - Ensures eventual consistency for rarely read data.
- **Limitations**:
    - Not all systems implement this (e.g., Voldemort lacks an anti-entropy process).

### **Quorums for Reading and Writing**:

- **Concept**:
    - Quorums are a mechanism to ensure consistency and fault tolerance in distributed systems, particularly in leaderless replication setups.
    - They define the minimum number of nodes that must participate in read and write operations to guarantee that the system returns up-to-date data.

### **Quorum Parameters**:

1. **n (Replication Factor)**:
    - The total number of replicas (nodes) in the system.
    - Example: If **n = 3**, the data is stored on 3 nodes.
2. **w (Write Quorum)**:
    - The minimum number of replicas that must acknowledge a write for it to be considered successful.
    - Example: If **w = 2**, at least 2 out of 3 replicas must confirm the write.
3. **r (Read Quorum)**:
    - The minimum number of replicas that must be queried during a read operation.
    - Example: If **r = 2**, the client reads from at least 2 out of 3 replicas.

### **Quorum Condition**:

- **Rule**:
    - To ensure consistency, the system must satisfy the condition:
        
        **w + r > n**
        
    - This guarantees that the sets of nodes involved in reads and writes overlap, ensuring at least one node has the latest data.
- **Example**:
    - For **n = 3**, setting **w = 2** and **r = 2** ensures that:
        - Every write is stored on at least 2 nodes.
        - Every read queries at least 2 nodes, ensuring at least one node has the latest value.

### **Fault Tolerance**:

- **How It Works**:
    - If **w < n**, the system can tolerate some node failures during writes.
    - If **r < n**, the system can tolerate some node failures during reads.
- **Examples**:
    1. **n = 3, w = 2, r = 2**:
        - Can tolerate **1 node failure**.
        - If one node is down, the system can still perform writes and reads using the remaining 2 nodes.
    2. **n = 5, w = 3, r = 3**:
        - Can tolerate **2 node failures**.
        - If two nodes are down, the system can still perform writes and reads using the remaining 3 nodes.

### **Trade-offs in Quorum Configuration**:

1. **Higher Write Quorum (w)**:
    - Increases durability (more replicas store the data).
    - Reduces write availability (more nodes must be available for writes to succeed).
2. **Higher Read Quorum (r)**:
    - Increases consistency (more replicas are checked for the latest data).
    - Reduces read performance (more nodes must respond).
3. **Lower Quorums (w and r)**:
    - Improves availability and performance but reduces consistency and durability.
- **Example Configurations**:
    - **Write-Heavy Workload**: Set **w = n** and **r = 1** to prioritize write durability.
    - **Read-Heavy Workload**: Set **w = 1** and **r = n** to prioritize read performance.

### **Quorum in Practice**:

- **Dynamo-Style Databases**:
    - Systems like Amazon Dynamo, Cassandra, and Riak use quorum-based replication.
    - Parameters **n**, **w**, and **r** are configurable to suit specific application needs.
- **Handling Failures**:
    - If fewer than **w** or **r** nodes are available, the system returns an error for writes or reads, respectively.
    - Failures can include node crashes, disk errors, or network interruptions.

### **Example Scenario**:

- **Setup**:
    - **n = 3** (data stored on 3 nodes).
    - **w = 2**, **r = 2** (quorum condition: 2 + 2 > 3).
- **Write Operation**:
    - Client sends a write to all 3 nodes.
    - At least 2 nodes must acknowledge the write for it to succeed.
- **Read Operation**:
    - Client reads from at least 2 nodes.
    - If one node is down, the client still gets the latest value from the other 2 nodes.
- **Node Failure**:
    - If one node fails, the system continues to function with the remaining 2 nodes.

## **Limitations of Quorum Consistency**:

- **Quorum Guarantees**:
    - When **w + r > n**, quorums generally ensure that reads return the most recent value, as the read and write sets overlap.
    - However, there are edge cases and practical limitations where stale values can still be returned.

### **Edge Cases and Limitations**:

1. **Sloppy Quorums**:
    - In systems using sloppy quorums (e.g., Dynamo-style databases), writes may land on different nodes than reads, breaking the overlap guarantee.
    - Example: A write might go to a temporary node during a network partition, and a read might not include that node.
2. **Concurrent Writes**:
    - If two writes occur concurrently, it’s unclear which one happened first.
    - Merging concurrent writes is necessary, but last-write-wins (LWW) based on timestamps can lead to data loss due to clock skew.
3. **Concurrent Reads and Writes**:
    - If a read happens during a write, it may return either the old or new value, depending on which replicas are queried.
4. **Partial Write Failures**:
    - If a write succeeds on some replicas but fails on others (e.g., due to disk full errors), the write is not rolled back on successful replicas.
    - Subsequent reads may or may not reflect the partially successful write.
5. **Node Failures and Data Restoration**:
    - If a node with a new value fails and is restored from a replica with an old value, the quorum condition (**w**) for the new value may no longer be met.
6. **Timing Issues**:
    - Even with correct configurations, timing issues can lead to stale reads (e.g., unlucky timing of read and write operations).

### **Quorum Configuration Trade-offs**:

- **w + r > n**:
    - Ensures strong consistency but may reduce availability and increase latency.
- **w + r ≤ n**:
    - Improves availability and latency but increases the likelihood of stale reads.
- **Majority Quorums**:
    - Setting **w** and **r** to a majority (e.g., **w = r = (n + 1) / 2**) balances consistency and fault tolerance.

### **Lack of Stronger Guarantees**:

- **Quorums Do Not Provide**:
    - **Read-your-writes consistency**: A user may not see their own writes immediately.
    - **Monotonic reads**: A user might see data moving "backward in time."
    - **Consistent prefix reads**: Writes may appear out of order.
- **Stronger Guarantees**:
    - Require transactions or consensus protocols (discussed in later chapters).

### **Monitoring Staleness**:

- **Leader-Based Replication**:
    - Replication lag is measurable, as writes follow a fixed order in the replication log.
    - Metrics can track the difference between the leader’s and followers’ positions in the log.
- **Leaderless Replication**:
    - Monitoring is more challenging due to the lack of a fixed write order.
    - Without anti-entropy, stale values can persist indefinitely for infrequently read data.
- **Research and Best Practices**:
    - Some research focuses on measuring staleness and predicting stale reads based on **n**, **w**, and **r**.

## **Sloppy Quorums and Hinted Handoff**:

- **Context**:
    - Quorum-based systems can tolerate node failures and slow nodes, but network interruptions can make it difficult to reach a quorum of the designated **n** nodes.
    - Sloppy quorums and hinted handoff are mechanisms to improve availability during such disruptions.

### **Sloppy Quorums**:

- **Definition**:
    - A sloppy quorum allows writes and reads to be served by nodes outside the designated **n** "home" nodes for a value.
    
    > Analogous to temporarily staying at a neighbor’s house when locked out of your own.
    > 
- **How It Works**:
    - During a network interruption, if the client cannot reach the usual **n** nodes, it writes to or reads from any reachable nodes.
    - These nodes temporarily store the data until the network is restored.
- **Benefits**:
    - Increases write availability: As long as **w** nodes are reachable, writes can succeed.
    - Maintains durability by ensuring data is stored on **w** nodes, even if they are not the designated ones.
- **Limitations**:
    - No guarantee that reads will return the latest value, as the latest write might be on a non-designated node.
    - Once the network is restored, hinted handoff ensures data is moved back to the designated nodes.
- **Usage**:
    - Enabled by default in Riak; optional in Cassandra and Voldemort.

### **Hinted Handoff**:

- **Definition**:
    - The process of transferring data from temporary nodes (used during a network interruption) back to the designated **n** nodes once the network is restored.
    
    > Analogous to returning home after staying at a neighbor’s house.
    > 
- **How It Works**:
    - Nodes that temporarily accepted writes on behalf of others forward the data to the appropriate "home" nodes.
    - Ensures data consistency and durability after a disruption.

### **Multi-Datacenter Operation**:

- **Leaderless Replication in Multi-Datacenter Setups**:
    - Leaderless replication is well-suited for multi-datacenter deployments due to its tolerance for network issues and concurrent writes.
- **Implementation in Cassandra and Voldemort**:
    - The total number of replicas (**n**) includes nodes across all datacenters.
    - Clients send writes to all replicas but wait for acknowledgments only from a quorum within their local datacenter.
    - Cross-datacenter writes are often asynchronous to avoid latency issues.
- **Implementation in Riak**:
    - Each datacenter operates as an independent cluster with its own **n** replicas.
    - Cross-datacenter replication happens asynchronously in the background, similar to multi-leader replication.

### **Trade-offs and Considerations**:

1. **Availability vs. Consistency**:
    - Sloppy quorums improve availability during network interruptions but weaken consistency guarantees.
    - Reads may return stale data until hinted handoff completes.
2. **Latency and Performance**:
    - Multi-datacenter setups prioritize local quorums to minimize latency, but cross-datacenter replication introduces delays.
3. **Configuration Flexibility**:
    - Systems like Cassandra and Voldemort allow fine-tuning of quorum settings and replication strategies for specific use cases.

### **Detecting Concurrent Writes**:

- **Context**:
    - In Dynamo-style databases, multiple clients can write to the same key concurrently, leading to conflicts.
    - Conflicts can arise during normal writes, read repair, or hinted handoff.
    - The order of writes may differ across replicas due to network delays or failures, causing inconsistencies.

### **Last Write Wins (LWW)**:

- **Mechanism**:
    - Each write is tagged with a timestamp or unique identifier.
    - The write with the highest timestamp "wins," and older writes are discarded.
- **Advantages**:
    - Ensures eventual convergence to a single value.
- **Disadvantages**:
    - **Data Loss**: Concurrent writes are discarded, even if they were successful.
    - **Clock Skew**: Timestamps may be unreliable due to clock synchronization issues.
    - **Durability Issues**: LWW is not suitable for systems where data loss is unacceptable.
- **Use Case**:
    - Acceptable in scenarios like caching, where lost writes are tolerable.

### **The "Happens-Before" Relationship**:

- **Definition**:
    - An operation **A** happens before operation **B** if **B** depends on or is aware of **A**.
    - If neither operation depends on the other, they are considered concurrent.
- **Determining Concurrency**:
    - Two operations are concurrent if neither happens before the other.
    - Example: Two clients writing to the same key without knowing about each other’s writes.
- **Algorithm for Detecting Concurrency**:
    1. Each key has a version number that increments with every write.
    2. Clients include the version number from their last read when writing.
    3. The server overwrites values with lower or equal version numbers but keeps higher versions as concurrent siblings.

### **Merging Concurrent Writes**:

- **Challenge**:
    - Concurrent writes create multiple versions (siblings) of the same key.
    - Clients must merge these siblings to resolve conflicts.
- **Merging Strategies**:
    1. **Union of Values**:
        - Combine all values from siblings (e.g., for a shopping cart, take the union of items).
    2. **Tombstones for Deletions**:
        - Use deletion markers (tombstones) to handle removed items during merges.
    3. **Conflict-Free Replicated Data Types (CRDTs)**:
        - Automatically merge siblings in a sensible way (e.g., Riak’s datatype support).
- **Example**:
    - Two clients concurrently update a shopping cart:
        - Client 1 adds milk and flour.
        - Client 2 adds eggs and ham.
        - The merged result includes all items: [milk, flour, eggs, ham].

### **Version Vectors**:

- **Purpose**:
    - Track dependencies and concurrency across multiple replicas.
- **Mechanism**:
    - Each replica maintains its own version number and tracks versions from other replicas.
    - The collection of version numbers is called a **version vector** (or **dotted version vector** in Riak).
- **How It Works**:
    - Version vectors are sent to clients during reads and returned during writes.
    - They help the database distinguish between overwrites and concurrent writes.
- **Benefits**:
    - Ensures no data is lost during replication.
    - Allows safe reads and writes across different replicas.

### **Conclusion**:

- **Concurrent Writes**:
    - Dynamo-style databases allow concurrent writes, leading to conflicts that must be resolved.
    - The **happens-before** relationship helps determine whether operations are concurrent or dependent.
- **Conflict Resolution**:
    - **Last Write Wins (LWW)**: Simple but prone to data loss.
    - **Merging Siblings**: Requires application logic or automated tools like CRDTs.
    - **Version Vectors**: Track dependencies across replicas to ensure consistency.
- **Best Practices**:
    - Use version vectors or CRDTs for robust conflict resolution.
    - Avoid LWW unless data loss is acceptable.
    - Design applications to handle sibling merging gracefully.

**Synchronous Replication**
• **Overview:** Synchronous replication ensures data consistency by requiring all followers to acknowledge a write before considering it successful, offering strong durability but potentially impacting availability.
• **Advantages:** Guarantees data consistency and durability.
• **Disadvantages:** Vulnerable to system halts if the synchronous follower fails.
• **Data Consistency:** Synchronous replication ensures that all replicas have the same data at any given time.
• **System Halts:** The system can halt if a synchronous follower fails because the leader waits for confirmation from all followers before acknowledging a write.
**Multi-Leader Replication**
• **Overview:** Multi-leader replication extends leader-based replication, enabling multiple nodes to accept writes and forward them to other leaders, improving performance and fault tolerance, particularly in multi-datacenter environments.
• **Use Cases:**
    ◦ Multi-Datacenter Operation:

        ▪ **Setup:** Each datacenter has its own leader, with asynchronous replication between datacenters.
        ▪ **Advantages:** Improved performance through local writes, tolerance of datacenter and network outages.
• **Challenges:**
    ◦ Replication Lag: Asynchronous replication can lead to significant lag, particularly in offline or collaborative scenarios.
    ◦ Write Conflicts: Concurrent modifications to the same data on different leaders require conflict resolution.
• **Write Conflicts:**
    ◦ **Problem:** Write conflicts occur when the same data is modified concurrently on different leaders, requiring conflict resolution.
    ◦ **Conflict Detection:** Conflicts are detected asynchronously in multi-leader systems.
    ◦ **Example:** Two users editing the same wiki page title simultaneously.
• **Conflict Resolution:**
    ◦ **Operational Transformation (OT):** Algorithm for collaborative editing (e.g., Google Docs, Etherpad).
    ◦ Automatic conflict resolution could simplify multi-leader replication for applications.
• **Topologies:**
    ◦ **Definition:** Define the paths through which writes are propagated between nodes.
    ◦ **Examples:**
        ▪ All-to-all: Highest resilience, but may face causality issues.
        ▪ Circular and Star: Simpler, but vulnerable to single points of failure.
• **Leaders and Followers in Replication:**
    ◦ **Leader (Master/Primary):** Handles all write requests from clients, writes data to local storage, and sends data changes to followers.
    ◦ **Followers (Read Replicas/Slaves/Secondaries):** Receive and apply changes from the leader in the same order.
**Node Outages**
• **Overview:** Node outages are a fundamental challenge in distributed systems, requiring mechanisms to maintain availability and data consistency during failures, whether due to crashes or planned maintenance. This involves strategies for follower recovery, leader failover, and managing potential data loss and inconsistencies.
• **Follower Failure - Catch-up Recovery:**
    ◦ Followers maintain a log of data changes from the leader.
    ◦ Upon recovery, the follower identifies the last processed transaction using its log.
    ◦ The follower requests all changes made during the outage from the leader.
    ◦ The follower applies these changes to catch up.
    ◦ The process can vary from fully automated to manual workflows.
• **Leader Failure - Failover:**
    ◦ **Process:**
        ▪ Detect Leader Failure: Use timeouts to identify unresponsive leaders.
        ▪ Choose a New Leader: Elect the replica with the most up-to-date data changes (consensus problem).
        ▪ Reconfigure the System: Redirect client writes to the new leader. Ensure the old leader becomes a follower upon rejoining.
    ◦ **Challenges:**
        ▪ **Data Loss in Asynchronous Replication:** Unreplicated writes may be discarded, potentially violating durability.
• **Data Loss:**
    ◦ Unreplicated writes from the old leader may be discarded, potentially violating durability expectations
• **Inconsistent State:**
    ◦ Occurs when data across systems becomes out of sync.
    ◦ **Example:** GitHub incident where an out-of-date follower reused primary keys, leading to inconsistencies between MySQL and Redis.
• **Split Brain:**
    ◦ Two nodes believe they are the leader.
    ◦ Leads to potential data loss or corruption if both accept writes.
• **Timeout Configuration:**
    ◦ Long timeouts delay recovery.
    ◦ Short timeouts can cause unnecessary failovers due to temporary issues.
• **Failover Trade-offs:**
    ◦ Automatic failover is complex and can be error-prone.
    ◦ Some teams prefer manual failover due to its potential for control.
**Replication**
• **Overview:** Replication involves maintaining data copies across multiple machines to enhance availability, reduce latency, and improve read throughput. It addresses the challenge of managing changes to replicated data, focusing on single-leader, multi-leader, and leaderless replication algorithms.
• **Challenges in Replication:** The primary difficulty lies in handling changes to replicated data. Key considerations include synchronous vs. asynchronous replication and handling failed replicas.
• **Synchronous vs. Asynchronous Replication:** Synchronous replication ensures data is written to multiple replicas before acknowledging the write, offering stronger consistency but potentially higher latency. Asynchronous replication allows the leader to acknowledge writes immediately, improving performance but introducing potential replication lag.
• **Historical Context:** Replication principles, rooted in 1970s research, remain relevant. Despite recent mainstream adoption of distributed databases, misconceptions around concepts like eventual consistency persist. Guarantees such as read-your-writes and monotonic reads are also discussed.
• **Leaders and Followers:**
    ◦ **Leader (Master/Primary):** Handles all write requests, writes new data to its local storage, and sends data changes to followers via a replication log or change stream.
    ◦ **Followers (Read Replicas/Slaves/Secondaries):** Receive and apply changes from the leader in the same order.
• **Replication Logs:** Replication logs are used to record data changes.

    ◦ **Statement-Based:** Simple but problematic for non-deterministic operations.
    ◦ **WAL Shipping:** Efficient but tightly coupled to storage engine.
    ◦ **Logical Logs:** Flexible and decoupled, enabling version independence and external use.
    ◦ **Trigger-Based:** Highly flexible but complex and resource-intensive.
• **Problems with Replication Lag:** Asynchronous replication can lead to replication lag. This can lead to temporary inconsistencies (eventual consistency).

    ◦ **Application-Level Solutions:** Implement stronger guarantees (e.g., read-after-write) in application code, though this adds complexity.
    ◦ **Database-Level Solutions:** Use transactions to provide stronger consistency guarantees.
**Quorum**
• **Overview:** Quorum is a mechanism used in distributed systems to maintain consistency and fault tolerance, especially in leaderless replication. It defines the minimum number of nodes required for read and write operations to ensure data integrity.
• **Quorum Parameters:**
    ◦ **n (Replication Factor):** Total number of replicas in the system.
    ◦ **w (Write Quorum):** Minimum replicas that must acknowledge a write for it to be successful.
    ◦ **r (Read Quorum):** Minimum replicas that must be read to consider a read operation successful.
• **Quorum Condition:**
    ◦ **w + r > n:** Ensures strong consistency but can impact availability and latency.
    ◦ **w + r ≤ n:** Improves availability and latency but increases the likelihood of stale reads.
    ◦ **Majority Quorums:** Setting w and r to a majority balances consistency and fault tolerance.
• **Fault Tolerance:**
    ◦ Systems can tolerate node failures and slow nodes.
    ◦ If one node fails, the system continues to function, provided the quorum condition is met.
• **Trade-offs in Quorum Configuration:**
    ◦ **Availability vs. Consistency:** Sloppy quorums improve availability but weaken consistency guarantees.
    ◦ **Latency and Performance:** Multi-datacenter setups prioritize local quorums to minimize latency.
    ◦ **Configuration Flexibility:** Systems like Cassandra and Voldemort allow fine-tuning of quorum settings.
• **Limitations of Quorum Consistency:**
    ◦ Quorums do not provide read-your-writes consistency or monotonic reads.
    ◦ Sloppy quorums can break the overlap guarantee, potentially leading to stale reads.
    ◦ Concurrent writes can cause conflicts, making it unclear which write happened first.
    ◦ Timing issues can lead to stale reads.
• **Sloppy Quorums:**
    ◦ Allow writes and reads to be served by nodes outside the designated "home" nodes.
    ◦ Improve availability during network interruptions.
    ◦ Can lead to stale data until hinted handoff completes.
• **Hinted Handoff:**
    ◦ A mechanism to improve availability during network disruptions.
• **Detecting Concurrent Writes:**
    ◦ Multiple clients can write concurrently, leading to conflicts.
    ◦ Conflicts can arise during normal writes, read repair, or hinted handoff.
• **Node Failures and Data Restoration:**
    ◦ If a node with a new value fails and is restored from a replica with an old value, the quorum condition (w) for the new value may no longer be met.
• **Anti-Entropy Process:**
    ◦ A background process that continuously compares replicas and copies missing data to ensure eventual consistency for rarely read data.
**Detecting Concurrent Writes**
• **Overview:** Detecting concurrent writes is crucial in distributed systems to handle conflicts arising from network delays, failures, or multiple clients updating the same data simultaneously. Various mechanisms are used to identify and resolve these conflicts, ensuring data consistency.
• **Last Write Wins (LWW):**
    ◦ **Mechanism:** Uses timestamps or unique identifiers to determine the most recent write.
    ◦ **Advantages:** Ensures eventual convergence to a single value.
    ◦ **Disadvantages:** Data loss due to discarding older writes, potential issues with clock skew, and unsuitability where data loss is unacceptable.
    ◦ **Use Case:** Suitable for scenarios like caching where some data loss is acceptable.
• **The "Happens-Before" Relationship:**
    ◦ **Definition:** Operation A "happens before" operation B if B depends on A. Otherwise, operations are concurrent.
    ◦ **Determining Concurrency:** Two operations are concurrent if neither happens before the other.
    ◦ **Algorithm for Detecting Concurrency:** Uses version numbers to track write order, helping servers identify concurrent writes.
• **Merging Concurrent Writes:**
    ◦ **Challenge:** Concurrent writes create multiple versions of the same data (siblings) that must be merged.
    ◦ **Merging Strategies:**
        ▪ **Union of Values:** Combines all values from siblings.
        ▪ **Tombstones for Deletions:** Uses markers to handle removed items during merges.
        ▪ **Conflict-Free Replicated Data Types (CRDTs):** Automatically merge siblings in a sensible way.
    ◦ **Example:** Merging a shopping cart with items added by different clients results in a combined cart containing all items.
• **Version Vectors:**
    ◦ **Purpose:** Track dependencies across replicas to ensure consistency.
    ◦ **Benefits:** Allows safe reads and writes across different replicas.
**Multi-Leader Replication Topologies**
• **Overview:** Multi-leader replication enables multiple nodes to accept writes, forwarding them to other leaders for high availability and performance, especially in multi-datacenter setups. Various topologies exist, each with distinct trade-offs regarding fault tolerance, complexity, and potential for conflicts.
• **All-to-All Topology:**
    ◦ Description: Every leader sends its writes to every other leader.
    ◦ Advantages: High fault tolerance.
    ◦ Disadvantages: Potential for causality issues if writes arrive out of order.
• **Circular Topology:**
    ◦ Description: Each node forwards writes to one other node in a circular manner.
    ◦ Advantages: Simpler than all-to-all.
    ◦ Disadvantages: A single node failure can disrupt replication. Requires manual reconfiguration.
• **Star Topology:**
    ◦ Description: A designated root node forwards writes to all other nodes.
    ◦ Advantages: Centralized control over replication flow.
    ◦ Disadvantages: The root node is a single point of failure.
• **Tree Topology:**
    ◦ Description: A generalization of the star topology, with hierarchical forwarding of writes.
    ◦ Advantages: Scalable for large systems.
    ◦ Disadvantages: Vulnerable to failures at higher levels of the hierarchy.
• **Challenges in Topologies:**
    ◦ Replication Loops:

        ▪ Solution: Tag writes with unique node identifiers.
    ◦ Fault Tolerance:

        ▪ Circular/Star Topologies: Vulnerable to single-node failures.
        ▪ All-to-All Topologies: More resilient due to multiple paths.
    ◦ Causality and Ordering:

        ▪ Problem: Writes may arrive out of order due to varying network speeds.
        ▪ Example: An update may arrive before the corresponding insert, violating causality.
        ▪ Solution: Use version vectors to track dependencies and ensure correct ordering.
• **Causality and Ordering:**
    ◦ Problem: Writes may arrive out of order due to varying network speeds.
    ◦ Example: An update may arrive before the corresponding insert, violating causality.
    ◦ Solution: Use version vectors to track dependencies and ensure correct ordering.
• **Implementation Issues:**
    ◦ Conflict Detection: Many systems have poor support for causal ordering and conflict detection.
    ◦ Recommendations: Carefully review documentation and test the system.
• **Writing to the Database When a Node Is Down:**
    ◦ Leaderless Configuration: No failover process exists. Clients send writes to all replicas in parallel.
• **Fault Tolerance:**
    ◦ All-to-all topologies offer the highest resilience.
    ◦ Circular and star topologies are simpler but vulnerable to single points of failure.
• **Replication Loops:**
    ◦ Tag writes with unique node identifiers to prevent infinite loops.
• **Use Cases for Multi-Leader Replication:**
    ◦ Multi-Datacenter Operation:

        ▪ Setup: Each datacenter has its own leader, with asynchronous replication.
        ▪ Advantages:

            • Performance: Writes are processed locally, reducing latency.
            • Tolerance of Datacenter Outages: Each datacenter operates independently.
            • Tolerance of Network Problems: Asynchronous replication handles temporary network interruptions better.
**Handling Write Conflicts**
• **Overview:** Handling write conflicts involves strategies to address inconsistencies that arise in distributed systems due to concurrent updates, network delays, or failures. The main goal is to maintain data integrity and ensure eventual consistency across multiple replicas, through conflict detection, avoidance, and resolution techniques.
• **Identifying Conflicts:**
    ◦ Obvious conflicts arise from concurrent writes to the same field within the same record.
    ◦ Subtle conflicts, like overlapping bookings in a meeting room system, need careful detection and resolution.
• **Conflict Avoidance:**
    ◦ Ensure all writes for a specific record go through the same leader.
    ◦ **Limitations:** Breaks down during leader rerouting (e.g., datacenter failure or user relocation).
• **Conflict Detection:**
    ◦ In multi-leader systems, conflicts are detected asynchronously.
    ◦ Synchronous conflict detection is not practical in multi-leader setups.
• **Conflict Resolution Strategies:**
    ◦ **Last Write Wins (LWW):**
        ▪ Use a timestamp or unique ID to pick the "winning" write.
        ▪ **Advantages:** Ensures eventual convergence to a single value.
        ▪ **Disadvantages:** Prone to data loss, clock skew issues, and durability issues.
        ▪ **Use Case:** Acceptable in scenarios like caching, where lost writes are tolerable.
    ◦ **Replica Priority:** Higher-priority replicas override lower-priority ones.
    ◦ **Value Merging:** Combine conflicting values (e.g., concatenate alphabetically).
    ◦ **Explicit Conflict Recording:** Store conflicts and resolve later (e.g., via user prompts).
• **Custom Conflict Resolution Logic:**
    ◦ **On Write:** Conflict handlers resolve conflicts immediately during replication (e.g., Bucardo).
    ◦ Runs in the background without user interaction.
    ◦ **On Read:** Conflicting versions are stored and returned to the application for resolution (e.g., CouchDB).
• **Automatic Conflict Resolution:**
    ◦ **Research and Techniques:**
        ▪ **Conflict-Free Replicated Data Types (CRDTs):** Data structures that automatically resolve conflicts.
        ▪ **Mergeable Persistent Data Structures:** Track history and use three-way merge functions.
        ▪ **Operational Transformation (OT):** Algorithm for collaborative editing.
    ◦ **Future Potential:** Could simplify multi-leader replication.
    ◦ **Challenges:** Custom resolution logic can be complex and error-prone.
• **Limitations**
    ◦ Conflict resolution is typically per row/document, not per transaction.
• **Converging Toward Consistency:**
    ◦ **Challenge:** Multi-leader systems lack a defined write order, leading to inconsistent states across replicas.
**Leader-Based Replication**
• **Overview:** Leader-based replication is a method used in distributed systems to maintain data consistency and availability. It involves a leader (master) node that handles write operations and followers (read replicas) that replicate the data.
• **Replica Roles:**
    ◦ **Leader (Master/Primary):** Handles all write requests, writes data to its local storage, and sends data changes to followers.
    ◦ **Followers (Read Replicas/Slaves/Secondaries):** Receive and apply changes from the leader, maintain a local copy of the database, and serve read requests.
• **Client Interaction:**
    ◦ **Writes:** Only accepted by the leader.
    ◦ **Reads:** Can be served by either the leader or any follower.
• **Applications:**
    ◦ **Databases:** Relational databases like PostgreSQL, MySQL, Oracle Data Guard, and SQL Server’s AlwaysOn Availability Groups. Nonrelational databases like MongoDB, RethinkDB, and Espresso.
    ◦ **Other Systems:** Distributed message brokers like Kafka and RabbitMQ, network filesystems, and replicated block devices like DRBD.
• **Synchronous vs. Asynchronous Replication:**
    ◦ *Note: The provided text does not explicitly define the differences between synchronous and asynchronous replication. However, the key consideration is mentioned.*
• **Setting Up New Followers:**
    ◦ *Note: The provided text does not explicitly provide the steps to setting up new followers. However, the following points can be inferred:*
    ◦ Create a snapshot of the leader's data.
    ◦ Request data changes from the leader's replication log since the snapshot.
    ◦ Process the backlog of changes to catch up and then handle real-time changes.
• **Handling Node Outages:**
    ◦ **Follower Failure - Catch-up Recovery:** Followers log data changes and use this log to identify and apply missing transactions upon recovery.
    ◦ **Leader Failure - Failover:**
        ▪ **Process:** Detect leader failure, choose a new leader (usually the replica with the most up-to-date data), and reconfigure the system to redirect client writes to the new leader.
        ▪ **Challenges:** Data loss in asynchronous replication.
• **Implementation of Replication Logs:**
    ◦ **Statement-Based Replication:** The leader logs every write request (e.g., INSERT, UPDATE, DELETE) and forwards the SQL statements to followers, which then execute them.
• **Timeout Configuration:**
    ◦ Long timeouts delay recovery. Short timeouts risk unnecessary failovers.
• **Failover Trade-offs:**
    ◦ Automatic failover is complex and error-prone. Some teams prefer manual failover.
