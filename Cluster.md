## Cluster

Consistency level
- Can vary for each request
- Levels
  - any
  - one
  - quorum (RF / 2) + 1
  - all

Consistent hashing
 - Partition – storage location on a row (table row)
 - Token – int value generated by hashing algorithm – identifies the location of partition in the cluster
   - +/- 2 to the 64 value range

Partitioners
- Murmur3Partitioner (default, best practice) – uniform distribution based on Murmur 3 hash
- RandomPartitioner – uniform distribution based on md5 hash
- ByteOrderedPartitioner (legacy only) – lexical distribution based on key bytes

Virtual nodes (vnodes)
- Multiple smaller primary range segments (virtual nodes) owned by each machine instead of one large range
- Available in Cassandra 1.2+
- default is 256 nodes per machine
- not applicable to nodes that combines Solr and Hadoop
- advantages
  - when new node is added, the tokens aren't unbalanced
  - machines bootstrap faster b/c token ranges are distributed
  - virtual node failure impact is spread across the cluster
  - token range assignment is automated
- You can have cluster use vnodes and regular nodes across DCs but not within a single DC.
- Enabled in cassandra.yaml: num_tokens – setting it to a value bigger than 1

Replication
- Advantage
  - Disaster recovery
  - Replicate data closer to the user for less latency
  - Workload segregation – can create node for reporting that doesn't impact end users
- Organization
  - node
  - rack: logical grouping of physically related nodes
  - data center: set of racks
- Gossip is used to communicate cluster topology
  - once per second, each node contacts 1 to 3 others, requesting and sharing updates
  - node states (heart beats), node locations
  - when a nod joins a cluster, it gossips with seed nodes that are specified in cassandra.yaml
    - assign the same seed node to each node in a data center
    - if more than one DC, include seed node from each DC

- The snitch
  - informs its partitioner of their node’s rack and DC topology
  - enables replication that avoids duplication in a rack – to prevent data loss risk from all replicas residing in a single rack
  - cassandra.yaml: endpoint_snitch: SimpleSnitch (various snitches available – typically GossipingPropertyFileSnitch used but Ec2Snitch available for AWS users, etc)
- cassandra-rackdc.properties
  - dc
  - rack
- keyspace
  - replication factor: how many to make of each partition
  - replication strategy: on which node should each replica get placed
    - SimpleStrategy (for learning only) – one factor for entire cluster. 3 is the recommended minimum. creates replica on nodes subsequent to the primary range node based on token.
    - NetworkTopologyStrategy (enables multiple DCs) – separate factor for each data center in cluster. distributes replicas across racks and data centers
    `  WITH REPLICATION = {‘class’:’NetworkTopologyStrategy’, ‘dc-east:2’, ‘dc-west:3’}`
      - doesn't simply create replica on nodes subsequent to the primary range node based on token. it distributes replicas across racks so not all replicas reside in a single rack.
      - for replication across DC, “remote coordinator” is picked and single replication communication is sent instead of multiple when multiple repcas to nodes are supposed to be copied over to other DC
      - ideally, you want to have the same number or a number divisible by the replication factor number (when replication factor is 3, have 3 racks or 9, 27, etc)
    - all partitions are replicas, there are no “originals”

- Hinted handoff
  - recovery mechanism for writes targeting offline nodes (known to be down or fails to acknowledge)
  - the coordinator can store a hinted handoff in `system.hints` table and it's replayed when the target comes back online
  - could be applied to a whole DC going down, too
  - configurable in cassandra.yaml
    - hinted_handoff_enabled (default: true)
    - max_hint_window_in_ms (default: 3 hours): after this consecutive outage period, the hints are no longer generated until target node comes back online. node offline for longer can be made consistent with “repair” or other ops
  - `nodetool getendpoints mykeyspace mytable $token_id`: token id can be looked up by “select token(id) from mytable”

Consistency level
  - write: how many nodes must acknowledge they received and wrote the request?
  - read: how many nodes must acknowledge by sending their most recent copy of the data?
  - tuning consistency
    - write all, read one – for system doing many reads
    - write quorum, read quorum
    - write one, read all – for system doing many writes
    - immediate consistency formula: (nodes_written + nodes_read) > replication_factor

Anti-entropy
- Cassandra tries to provide consistency at read time
- As part of read `digest query` is sent to replica nodes and nodes with stale data are updated
- digest query: returns a hash to check the current data state
- read_repair_chance: set as table property value between 0 and 1, to set the probability with which read repairs should be invoked on non-quorum reads – fixes data for the next query
- nodetool repair
  - manual repair with `nodetool repair` is the last line of defense – might want to run every 7 days
  - repair = synchronizing data
  - when to run it
  - recovering failed node
  - bringing a down node back up
  - periodically on nodes with infrequent read data
  - periodically on nodes with write or delete activity
- if run periodically, do so at least every `gc_grace_seconds`
  - tombstone garbage collection periods (default: 864000 = 10 days)
  - tombstone is a marker placed on a deleted column within a partition. failure to repair within the period can lead to deleted column resurrection
- incremental repair – only repair new data – data that was introduced the system since the last repair
  - `tools/bin/sstablemetadata` and `sstablerepairedset` used for enabling incremental repair if you were using 2.0. no need if you start w/ 2.1.
  - recommended if using leveled compaction
  - run daily
  - avoid anticompaction by not using a partitioner range or subrange