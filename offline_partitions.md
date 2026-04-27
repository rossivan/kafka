# what are offline partitions?
* each kafka topic is split into partitions, and each partition has:
* one leader (handles reads/writes)
* zero or more followers (replicas)
* a partition becomes offline when:
** there is no leader assigned, or
** the leader is unreachable / unavailable
* offline partitions are partitions that currently have no available leader broker,
** which means producers and consumers cannot read from or write to them.
** they are a critical health indicator of your cluster


# detection, analysis and resolution


## detection - monitor offline partitions

### 1.  JMX metrics (most common)
* kafka.controller:type=KafkaController,name=OfflinePartitionsCount > 0 (you have a problem)

### 2. kafka CLI tools
* 'kafka-topic' CLI tool outputs 'Leader: -1' then partition is offline

### 3. kafka logs
* check for leader election failures, ISR shrink events, replica fetch issues

### 4. alerting best practices 
* OfflinePartitionsCount > 0 (critical alert)
* ISR count drops below threshold
* URP increase


## root-cause analysis

### 1. broker failures
* If a broker hosting partition leaders crashes or becomes unreachable, partitions it leads can go offline—especially if replicas can’t take over

### 2. insufficient in-sync replicas (ISR)
* Kafka requires a minimum number of replicas (min.insync.replicas) to elect a leader
* if too many replicas fall out of sync: implies no eligible leader → hence partition goes offline

### 3. unclean leader election disabled
* if unclean.leader.election.enable=false (common in production)
* kafka refuses to elect an out-of-sync replica as leader, causing downtime instead of risking data loss

### 4. network partitions
* cluster nodes unable to communicate (firewalls, latency, DNS issues) can prevent leader election

### 5. disk failures
* if a broker loses access to its log directories, partitions on that broker may go offline.

### 6. controller issues
* the Kafka controller (responsible for leader election) may fail or be slow to respond.

### 7. misconfiguration or maintenance
* rolling restarts done incorrectly
* all replicas of a partition placed on unavailable brokers
* misconfigured replication factor


# resolution

## immediate triage

### 1. check broker health
* restart failed brokers
* verify resource are not maxed out - cpu, disk, memory, network

### 2. check ISR status
* if ISR is too small, investigate lagging replicas

### 3. verify connectivity
* ensure brokers can reach other
* check firewall / DNS / network latency


## recovery actions

### 1. bring brokers back online
* most cases resolve automatically once brokers recover

### 2. reassign partitions
* if replicas are poorly distributed run 'kafka-reassign-partitions'

### 3. enable unclean leader election (last resort)
* configure broker with 'unclean.leader.election.enable=true'
* this may cause data loss, so only use in emergencies

### 4. increase replication factor
* if replication factor is too low (e.g., 1), you have no failover protection

### 5. fix disk issues
* replace failed disks
* ensure log.dirs are accessible

### 5. controller troubleshooting
* ensure controller broker is stable
* in newer Kafka versions (KRaft mode), check quorum health


# Prevention best practices
* use replication factor ≥ 3
* set appropriate min.insync.replicas (e.g., 2)
* monitor ISR shrinkage trends
* perform rolling restarts carefully
* distribute replicas across racks (rack awareness) - applicable for MRC cluster
* use automated rebalancing tools