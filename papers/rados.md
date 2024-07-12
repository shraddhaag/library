# RADOS: A Scalable, Reliable Storage Service for Petabyte-scale Storage Clusters

Paper: https://ceph.com/assets/pdfs/weil-rados-pdsw07.pdf 

## Cluster Management 

Each cluster has: 
* OSDs - large number, each has - RAM, CPU, network interface, local disk drive/RAID
* monitors - small number, responsible for managing OSDs cluster membership, standalone process with small amount of local storage

### Cluster Map
* Exclusively manages the RADOS cluster
* Completely specifies data distribution in a RADOS cluster 
* Manipulated by monitor cluster 
* Replicated by all OSDs and all clients

Each cluster map has the following information: 
|Information|Description|
|-----------|-----------|
|Map epoch|map version |
|OSDs that are up and down|network address of all healthy & unhealthy nodes|
|OSDs that are in and out|nodes included in PGs and nodes that are not. |
|Number of PGs||
|CRUSH|placement rules to map each PG to OSDs|

* every custer map change increments the epoch. 
* in a large cluster where map changes are more frequent, incremental maps (difference between two maps) is distributed. 

Possible state of OSDs: 
|OSD State|Description|
|---------|-----------|
|UP - IN|Actively serving I/O|
|DOWN - OUT|Failed|
|UP - OUT|Online but idle. eg - newly added devices which aren't immediately used.|
|DOWN - IN|Unreachable but data hasn't yet been migrated off. eg - intermittently unreachable, data migration in progress before decommissioning.|

### Map Propagation

* OSDs are responsible for sharing map updates along with inter OSD communication + heartbeat messages.
* map updates are shared between communicating OSDs (ie the ones sharing PGs), similar to gossip protocol
* each OSD does: 
    - history of cluster map updates recieved 
    - tags all its messages with the latest map epoch
    - latest epoch observed at all its peers 
* when contacting a peer with an old epoch, incremental updates are shared to bring it up to speed.
* a new update is shared across the cluster in O(log n) time.
* conservatively sharing updates, OSD can receive a maximum of x copy of updates where x = number of PGs it contains. 
* updates lazily propagated, OSDs receive updates as they interact with the cluster nodes. 

#### OSD boots up 
* informs monitor its up with its latest epoch 
* monitor sets the OSD to up and shares incremental updates 
* OSD starts communicating with peers:
    - bringing them up to date
    - also to maintain their latest epoch observed

### Data Placement 

* no large centralised allocation table. 
* objects are pseudo randomly assigned to devices. 
* objects are first assigned to to PGs and then each PG is placed on r OSDs. 

#### Step 1: Map Object to Placement Groups 
* placement groups are logical collection of objects 
* objects are mapped to a placement group using: 
    - hash of object name 
    - level of replication 
    - bit mask of total number of placement groups in the system 
* as system scales, total number of PGs are increased gradually 

||Mirroring|Complete Declustering|PGs (Partial Declustering)|
|-|-------------------------|---------|---------------------|
|Relication when device failure|||parallel recovery, replication peers related to number of PGs an OSD stores.|
|Coincident device failure|data loss chances are high|low chances of data loss|limited risk|

#### Step 2: Map PGs to OSDs 
* each PG is assigned to r OSDs based on the cluster map
* this mapping is done using CRUSH 

##### CRUSH 
* a robust replica distribution algorithm, similar to a hash function  
* calculates a stable, pseduo random mapping 
* when OSDs join or leave cluster, most PGs aren't moved. Just enough data is moved to maintain a balanced distribution. 
* also uses weights to control amount of data assigned to each device based on capacity & performance. 

## Replication 
* performed by OSDs 
* utilises inter cluster network bandwidth and simplifies things for client 
* serializability is preserved.
![three types of replication](/papers/rados-replication.png)

||Primary-Copy|Chain|Splay|
|-|-----------|-----|-----|
|Reads|Primary|Tail|Tail|
|Writes|Primary|Primary|Primary|
|Replication|Parallel|Serial|Parallel|

## Strong Consistency 
* all messages are tagged with the sender's cluster map epoch 
* request send to out of date OSD - OSD sends incremental updates back to client 
* master copy changes PG membership but its not propagated to any pariticipating OSD: 
    - updates are processed by old members till none of them have heard about the change 
    - new PG's primary OSDs responsible to contact old OSDs to update them with the new map + verify contents of PG 
    - above ensure, before new OSDs take over, old OSDs are informed to stop servicing I/O 
    - to prevent stale reads, an OSD can serve reads if it has received timely hearbeat messages from its peers with the same PG. If not, reads are blocked. 

## Failure Detection 
* TCP socket failure is detected if limited number of reconnection attempts fail. Failure is reported to monitor cluster.
* OSD failure is detected through missed heartbeat messages. If an OSD detects it has been marked down, it kills itself.  

## Data Migration 
* RADOS assumes no data distribution continuity between 2 map epochs
* OSDs aggressively maintains PG metadata, ie: 
    - PG log 
    - object version it contains (even when object itself is missing)
* peering algorithm to maintain a consistent view of PG contents

### Peering 
- when OSD receives an updated cluster map: 
    - incrementally walks through all recent map updates to update PG state 
    - if active list of OSDs changed: must re-peer with all OSDs listed
- driven by primary OSD of the PG. 
- all replica OSDs of a PG must send notify messages to primary with OSD state, PG log, latest epoch for last peering - this informs the primary of its role. 
- primary creates a prior set - all participating OSDs since last successful peering 
- primary then updates all active replicas by requesting fragments of PG log from prior OSDs if needed. 

### Recovery 
- parallelised thanks to declustered replication 
- performance critical: seeking and reading (ie this is the thing to minimise)

- if objects can be read from any participating OSD in a PG: 
    - duplicating reading the same object ie performance hits 
    - replication will become harder if an older version of an object is read.

- coordinated by the primary OSD in a PG. 
- all operations on a object are suspended if the primary does not have a copy. 
- if primary has a copy: it knows which replicas need it from the peering process and sends them a copy. 
- all objects are read only once. 

## Monitor Cluster 
- makes use of Paxos. 
- CP, not AP. 
- responsible for maintaining the master copy of cluster map. 
- majority of cluster map must be online to read/update the cluster map. 

### Paxos Service 
- initally a leader is elected to serialise map updates and maintain consistency. 
    - a probe to fetch the latest epoch of all other monitors is send which must be responded to in time T. 
    - if majority of monitors respond and a quorum is established:
        - the latest communicated epoch is sent to all active monitors. 
        - short term leases are distributed. 
- updates are forwarded to leader monitor: aggregates them and periodically udpates map by incrementing the epoch
    - distributes the new epoch to active monitors and revokes existing leases 
    - if majority acknowledge, the new map change is committed and new leases are issued. 

- leases give mintors ability to distribute cluster map to clients and OSDs. 
- an election is called when: 
    - if lease expires which getting renewed, leader is understood to be unreachable and election is called.
    - if leases are not acknowledged back to leader, previously active monitor is assumed to be dead and an election is called. 
    - a new monitor boots us / previous election did not complete within a reasonable time, an election is called.  

### Workload on monitors 

- distribution of map updates among OSDs is generally handled by OSDs themselves. 
- leases allow all active montiors to service reads from clients and rarely OSDs. 
- updates are handled by leader alone, which in turn aggregates so that map updates are tunable and independent on size of cluster.
- maximum number of possible failure report load on leader monitor: 
    - if each OSD stores X PGs, and Y OSDs fail: load is propotional to XY. 
- to prevent such a deluge: 
    - OSDs do heartbeat check semi randomly, so failure reports are staggered. 
    - reports are forwarded to monitor cluster: throttles and batched proptional to the cluster size.
    - active monitors forward these reports to leader only once. 
- final maximum load on leader monitor = YM where M is size of monitor cluster. 

## Glossary 
* RADOS - Reliable Autonomic Distributed Object Store 
* OSD - Object Storage Device 
* PG - Placement Groups 

## Questions 
- how is CRUSH better than a hash function? 
- has there ever been a breaking change to OSD or monitor software?
- how is deployment done? 