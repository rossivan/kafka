# what is “cluster-wide bits per second”?
* “cluster-wide bits per second” refers to the total network throughput across the entire Kafka cluster, measured in bits per second (bps).


# What it actually means?
* it’s the aggregate rate of data moving in and out of all brokers combined. This typically includes:
** incoming traffic (ingress): data produced to kafka by producers
** outgoing traffic (egress): data consumed by consumers
** replication traffic: data copied between brokers to keep partitions in sync


# simple way to think about it
* if you have multiple brokers:
** broker 1: 200 Mbps in + 300 Mbps out
** broker 2: 150 Mbps in + 250 Mbps out
** broker 3: 100 Mbps in + 200 Mbps out
** then the cluster-wide bits/sec would roughly be the sum of all those flows: → total throughput across the whole system


# Why it matters
* this metric is important for:
** capacity planning → are your network and brokers handling the load?
** bottleneck detection → spikes might indicate issues
** scaling decisions → when to add brokers or partitions
** cost awareness → especially in cloud environments with bandwidth pricing


# Where you see it
* you’ll typically encounter this metric in:
** monitoring tools (e.g., Prometheus + Grafana dashboards)
** Kafka metrics like:
*** BytesInPerSec
*** BytesOutPerSec
*** ReplicationBytesInPerSec
** these are often converted to bits/sec and summed across brokers to show a cluster-wide view.



# what is “cluster-wide Bytes Out per second”?
* “cluster-wide Bytes Out per second” means the total amount of data leaving the kafka cluster every second, measured in bytes/sec, summed across all brokers.


# What counts as “Bytes Out”
* this metric includes all data kafka is sending out, mainly:
** consumer fetch traffic → data delivered to consumers reading from topics
** follower replication fetches → when brokers replicate data from leaders to stay in sync
* so it’s not just external clients—internal replication traffic is included too.


# Cluster-wide = aggregated
* each broker has its own BytesOutPerSec.
* “Cluster-wide” just means: sum of BytesOutPerSec across every broker in the cluster
* quick example
* broker A: 50 MB/s
* broker B: 70 MB/s
* broker C: 30 MB/s
* → cluster-wide Bytes Out = 150 MB/s


# Why it matters
* this metric tells you:
** how much data consumers are pulling (read load)
** network utilization on brokers
** replication overhead (can be significant with high replication factors)
** when you may need to scale consumers or brokers


# common confusion
* bytes in per sec = data coming into Kafka (from producers)
* bytes out per sec = data leaving Kafka (to consumers + replication)



# what is “cluster-wide messages in per second”?
* “cluster-wide messages in per second” is the total number of kafka records being produced into the entire cluster every second, summed across all brokers.


# What it measures
* it counts: how many individual messages (records/events) are written to Kafka topics per second across the whole cluster
* each message is one event sent by a producer to a partition.


# why it’s “cluster-wide”
* kafka itself tracks this per broker (not as a single native cluster metric). So “cluster-wide” means:
** take MessagesInPerSec from each broker
** sum them across all brokers
* that gives the total event ingestion rate for the cluster.
* simple example:
** broker 1: 8,000 msg/sec
** broker 2: 12,000 msg/sec
** broker 3: 5,000 msg/sec
** → cluster-wide Messages In/sec = 25,000 msg/sec


# What it does not measure
* this metric is not about data size or performance directly, so it does NOT tell you:
** message payload size (that’s bytes in/sec)
** network bandwidth usage
** storage volume
* it only measures event frequency (rate of records).


# where it comes from
* in monitoring systems like Prometheus and dashboards like Grafana, it’s typically derived from Kafka broker JMX metrics such as:
** MessagesInPerSec
** aggregated across all brokers


# why it matters
* this metric is useful for:
** understanding event load (how “chatty” the system is)
** scaling decisions (more partitions or brokers if rate grows)
** detecting traffic spikes
** capacity planning for consumers and processing systems


# how it differs from related metrics
* Messages In/sec → number of records produced
* Bytes In/sec → total data volume of those records
* Bytes Out/sec → data sent to consumers + replication


# a key insight:
* high messages/sec + low bytes/sec → many small events
* low messages/sec + high bytes/sec → fewer large events



# what is “cluster-wide fetch session eviction per second”?
* “cluster-wide fetch session eviction per second” tracks how often fetch sessions are being evicted across the entire cluster per second.


# what is a “fetch session”?
* kafka uses fetch sessions as an optimization between brokers and consumers (or follower replicas).
* instead of sending full partition data every time, the broker and client maintain a session so only changes are transmitted.
* this reduces network and CPU overhead.


# what does “eviction” mean?
* a fetch session gets evicted (i.e., dropped) when:
** it becomes inactive (client stopped fetching)
** the broker runs out of space in its fetch session cache
** the session is explicitly closed or times out
** so the metric means: FetchSessionEvictionPerSec = number of fetch sessions removed per second (aggregated across all brokers in the cluster).


# how to interpret it
* low / near zero → normal and healthy
* moderate but stable → usually fine (some churn is expected)
* high or spiking → something to investigate:
** consumers reconnecting frequently
** short-lived consumers (e.g., poorly configured clients)
** fetch session cache too small
** network instability
** rolling broker restarts


# why it matters
* high eviction rates reduce the efficiency of kafka’s incremental fetch mechanism, which can:
** increase CPU usage on brokers
** increase network traffic
** add latency for consumers
** related metrics worth checking
* to get context, you’d usually look at:
** FetchSessionCacheHitRatio
** FetchSessionCacheMissRatio
** FetchSessionCount