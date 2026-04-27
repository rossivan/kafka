# why URPs matter?
* reduced fault tolerance if another replica fails, there is data loss risk
* producer failures if using acks=all and ISR < min.insync replicas, then writes fail
* potential unavailability if leader fails and no ISR partition becomes unavailable


# detection, analysis and resolution


## detection
### 1. monitor broker-level metrics - UnderReplicatedPartitions - via
* jmx
* prometheus + grafana
* c3 (confluent control center)

if urp > 0 -> something is wrong; if urp increasing -> active degradation

### 2. list URPs from a controllers view
### if broker came up recently and URPs are shrinking, that is normal catch-up behavior
kafka-topics    \
    --describe  \
    --bootstrap-server  kafka-1:9092    \
    --under-replicated-partitions

watch -n 5 'kafka-topics    \
    --describe  \
    --bootstrap-server  kafka-1:9092    \
    --under-replicated-partitions'

### 3. check parition state for affected topic(s) (look for isr < replicas)
kafka-topics    \
    --describe  \
    --topic topic-1 \
    --bootstrap-server  kafka-1:9092

### 4. inspect broker logs for
* ISR shrink / expand events
* follower lag warnings
* replica fetch failures


## root cause analysis - under-replication is almost always a symptom, not the root problem
### 1. broker failure
-- broker down or unreachable. entire broker missing from ISR
-- network partition. many partitions affected

### 2. slow follower (replica lag) - followers must fetch data from the leader fast enough
* disk I/O bottleneck
* cpu saturation
* network throughput limits
NOTE: kafka removes slow replicas from the isr if they exceed replica.lag.time.max.ms

### 3. disk issues
* high disk latency
* full disks
* disk throttling in cloud environments

### 4. network issues
* high latency between brokers
* packet loss
* cross-AZ bandwidth limits

### 5. large traffic spikes
* producers overwhelm brokers
* follower's can't keep up

### 6. misconfiguration
* replication factor too high for cluster size
* num.replica.fetchers too low
* improper min.insync.replicas

### 7. leader imbalance
* too many leaders on one broker
* that broker becomes overloaded

### 8. long garbage collection pauses


## resolution strategies

### 1. immediate triage
* check broker health. are any brokers down? restart if needed
* check ISR recovery. kafka auto-recovers if issue is transient

### 2. fix lagging replicas
* increase num.replica.fetchers
* increase replica.fetch.max.bytes
* increase replica.fetch.wait.max.ms

### 3. reduce pressure
* throttle producers
* pause heavy consumers (if impacting disk)

### 4. fix infrastructure bottlenecks
* disk - move to faster disks (SSD/NVMe). increase disk throughput limits
* network - ensure low latency between brokers. avoid cross-region replication unless tuned
* also verify advertised.listeners and inter-broker listener settings are correct. advertised.listeners is what brokers and clients use to reach a broker, bad advertised addresses are a classic cause of partial cluster connectivity
ss -tanp | grep 9092
ping <peer-broker-host>
traceroute <peer-broker-host>

### 5. rebalance leadership
* confluent-rebalancer moves leaders and data as well
confluent-rebalancer execute    \
    --bootstrap-server  kafka-1:9092    \
    --metrics-bootstrap-server  kafka-1:9092 \
    --throttle    1000000   \
    --verbose

* once all replicas are back in ISR consider a leader election after recovery.
* do this only after a cluster is healthy not while a broker is still lagging
* NOTE: kafka-leader-election only moves leaders
kafka-leader-election.sh \
    --bootstrap-server localhost:9092 \
    --election-type PREFERRED \
    --all-topic-partitions

### 6. reassign partitions
kafka-reassign-partitions.sh    \
    --bootstrap-server  server-1:39094    \
    --reassignment-json-file    reassign-topic-1.json   \
    --execute

### 7. adjust configuration carefully
* increase replica.lag.time.max.ms giving follower replicas more time to catch-up with leader
* beware though as higher values delay failure detection

### 8. last resort: remove bad replicas
* if a broker is permanently degraded decommission broker and reassign partitions to healthy nodes
* in this case URPs will not clear, so reassign partitions or replace.
* this would mean generating and applying a partition reassignment plan. tooling depends on kafka version but decision point is
** broker dead for good - replace broker or move replicas
** disk dead - move partitions off affected log dir / broker
** chronic hotspot - rebalance partitions


## Check topic durability settings before changing anything

### Need to determine if producers are already at risk of write failures or weaker durability
### focus on min.insync.replicas and unclean.leader.election.enable
### min.insync.replicas works with producer acks=all to enforce durability, and if the minimum cannot be met,
### producers can get NotEnoughReplicas or NotEnoughReplicasAfterAppend.
### with unclean.leader.election.enable a non-ISR replica could become a leader, which risks data loss

### 1. inspect suspected broker overrides
kafka-configs   \
    --bootstrap-server  kafka-1:9092    \
    --entity-type   brokers \
    --entity-default    \
    --describe

kafka-configs   \
    --bootstrap-server  kafka-1:9092    \
    --entity-type   brokers \
    --entity-name   5   \
    --describe

### 2. Inspect topic durability settings
kafka-configs   \
    --bootstrap-server  kafka-1:9092    \
    --entity-type   topics  \
    --entity-name   my-topic    \
    --describe


## check disk and log directory health

### a very common URP root cause is a broker being alive but unable to keep up because a log directory is slow or broken

### 1. useful OS checks on the broker
df -h
iostat -xz 1 5
dmesg | tail -100


### 2. Inspect log dirs on the suspect broker
kafka-log-dirs  \
    --bootstrap-server localhost:9092   \
    --describe  \
    --broker-list 3


# best practices
* keep UnderReplicatedPartitions = 0 in steady state
* alerton in URP > 0 for more than a few minutes
* use RF >= 3 in production
* balance partitions across brokers
* monitor disk I/O, network throughput and request latency