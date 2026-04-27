# what is "ActiveControllerCount"?
* ActiveControllerCount metric tells you how many brokers in the cluster currently believe they are the controller.


# What is a “controller” in Kafka?
* kafka clusters elect one broker to act as the controller. This broker is responsible for:
** managing partition leadership
** handling broker joins/leaves
** coordinating metadata updates
* at any given time, there should be exactly one active controller.


# what does ActiveControllerCount mean?
* Value = 1 → ✅ Healthy cluster (exactly one controller)
* Value = 0 → ❌ No controller (cluster is in trouble, likely unavailable)
* Value > 1 → ❌ Multiple controllers (split-brain situation, very bad)


# why it matters
* This metric is critical because:
** kafka depends on a single controller for consistency
** if no controller exists, cluster operations stall
** if multiple controllers exist, metadata corruption or instability can occur


# where you’ll see it
* this metric is usually exposed via:
** JMX (kafka.controller:type=KafkaController,name=ActiveControllerCount)
** Monitoring tools like Prometheus, Datadog, etc.


# practical takeaway
* if you're monitoring Kafka:
** alert if ActiveControllerCount ≠ 1
** it’s one of the fastest indicators of cluster health issues




# what is ActiveBrokerCount?
* ActiveBrokerCount is a controller-side metric that tells you how many brokers in the cluster are currently alive and participating.


# what it means?
* a broker is a kafka server that stores data and serves client requests.
* ActiveBrokerCount = number of brokers that are up, reachable, and registered with the cluster controller.
* it reflects the current health and size of your cluster from the controller’s perspective.


# why it matters?
* cluster health check: if this number drops unexpectedly, it usually means one or more brokers are down or disconnected.
* capacity & availability: fewer active brokers → less capacity and potential risk to partition replication.
* alerting signal: often used in monitoring systems to trigger alerts when brokers fail.
* example: you have 5 brokers configured.
** normally: ActiveBrokerCount = 5
** if one crashes: ActiveBrokerCount = 4


# related metrics
* OfflinePartitionsCount → partitions without a leader
* UnderReplicatedPartitions → replication not meeting the configured factor
* ControllerCount → should typically be 1 (only one active controller)




# what is TimedOutBrokerHeartbeatCount?
* TimedOutBrokerHeartbeatCount is a controller-side metric that tracks how many times a broker’s heartbeat failed to arrive within the expected time window.


# What it means?
* kafka brokers periodically send heartbeats to the controller to signal they’re alive.
* TimedOutBrokerHeartbeatCount increments when:
** a broker misses its heartbeat deadline, and
** the controller considers that heartbeat timed out
* this doesn’t always mean the broker is fully down. it means the controller temporarily lost timely communication with it.


# Why it matters
* early warning signal: spikes can indicate network latency, GC pauses, CPU pressure, or overloaded brokers.
* cluster instability hint: frequent timeouts can lead to brokers being marked offline, triggering leader elections and partition movement.
* precursor to bigger issues: if it continues, you may soon see drops in ActiveBrokerCount or increases in under-replicated partitions.


# common causes
* network hiccups or high latency between brokers and controller
* long JVM garbage collection pauses
* CPU or disk I/O saturation on brokers
* controller overload (especially in large clusters)
* misconfigured timeouts (too aggressive heartbeat/session settings)


# how to interpret it
* low, occasional increments: normal in many clusters
* frequent or rapidly increasing count: something is wrong and worth investigating