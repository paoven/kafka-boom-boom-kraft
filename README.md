# Boom Boom Kafka!! 

This project is an attempt to simulate network partition within a Kafka cluster and observe the behavior of the cluster.
The purpose is to evaluate the durability guarantees provided by a Kafka cluster in case of unreliable network.
Those scenarios try to emulate a Kafka stretch cluster over 2 datacenters.

## Most interesting results 

* Kafka can operate properly without Zookeeper as long as there is no need to change the ISR
  - The ISR update fails without Zookeeper and thus producer using _acks=all_ would be blocked by _out of sync_ members 
- It's possible to have multiple leaders for a single partition in case of network partition, but:
  - Only one leader can accept write with _acks=all
  - Using _acks=1_ could lead to data getting truncated
- To ensure durability of messages:
  - _acks_ must be set to _all_, otherwise you might write to an _invalid_ leader
  - _min.insync.replicas_ must be set up to ensure that data is replicated on all desired physical location
- If Zookeeper quorum is rebuilt, there is actually no guarantee that the new quorum have all the latest data:
  - this could results in data loss or in messages getting duplicated inside leaders 
	- it's probably better to rely on hierarchical quorum to avoid those issues

## Scenarios

### Scenario 1 - Leader is isolated

3 Kafka brokers: kafka-1, kafka-2 and kafka-3 and one zookeeper.
 
Current leader is on kafka-1 then kafka-1 blocks all incoming messages from kafka-2, kafka-3 and zookeeper

### Scenario 2 - Network split 1 ZK and 1 broker

4 Kafka brokers: kafka-1, kafka-2, kafka-3 and kafka-4 and one zookeeper.
 
Current leader is currently on kafka-1 then a network partition is simulated:
- on one side kafka-1, kafka-2 and kafka-3
- on another side kafka-4 and zookeeper 

### Scenario 3 - Rebuild the ZK quorum after a network partition

4 Kafka brokers: kafka-1, kafka-2, kafka-3 and kafka-4 and 3 zookeeper.
 
For some reasons, you decide to rebuild the quorum of zookeeper (e.g. you lost a rack or a DC).
 
There is no guarantee, after rebuilding a quorum, that the nodes have all the required information. 

### Scenario 4 - Complete network outage

4 Kafka brokers: kafka-1, kafka-2, kafka-3 and kafka-4 and 3 zookeeper.
 
Simulate a complete network outage between each and every component.

When the network comes back the quorum is reformed, and the cluster is healthy.

### Scenario 5 - DC network split

Network setup:
* DC-A: kafka-1, kafka-2, ZK-1, ZK-2
* DC-B: kafka-3, kafka-4, ZK-3
 
We simulate a DC network split.

When the network comes back the quorum is reformed, and the cluster is healthy.

### Scenario 6 - Broker connection loss

Network setup:
* DC-A: ZK-1, Kafka-1
* DC-B: ZK-2, Kafka-2
* DC-C: ZK-3, Kafka-3

We simulate the following connectivity loss:
* Kafka-1 --> X Kafka-3
* Kafka-2 --> X Kafka-3

All other connections are still up.
All partitions where Kafka-3 is the leader are unavailable.
If we stop Kafka-3, they are still unavailable as unclean leader election is not enabled and Kafka-3 is the only broker in ISR.


### Scenario 7 Kraft - DC network split

Network setup:
* DC-A: kafka-1, kafka-2, kraft-1
* DC-B: kafka-3, kafka-4, kraft-2
* DC-C kraft-3

We simulate a DC network split between kraft controllers of DC-A and DC-B but they can still see kraft-3

The goal was to observe that quorum leadership does not flap between kraft-1 and 2, in fact is stabilizes to kraft-3 during the network split

When the network comes back the quorum is reformed, both kraft-1 and kraft-2 can become again leaders

### Scenario 8 Kraft - 5 controllers - partial cross DC network split described in KIP-996

Network setup:
* DC-A: kafka-1, kafka-2, kraft-1, kraft-4 (id=8)
* DC-B: kafka-3, kafka-4, kraft-2, kraft-5 (id=9)
* DC-C kraft-3

We simulate a particular case of DC network split between some of the kraft controllers of DC-A and DC-B as reported in [KIP-996]
(https://cwiki.apache.org/confluence/display/KAFKA/KIP-996:+Pre-Vote) 

![enter image description here](https://decentralizedthoughts.github.io/uploads/RAFT%201.jpg)

During the network outage, we can observe that the quorum keeps flapping between the connected nodes (1,2,3 with reference to the image)

When the network comes back the quorum is reformed and stable

KIP-996 should avoid instability, created this failure scenario to simplify testing once it will be available
