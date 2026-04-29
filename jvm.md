# what is jvmMemory metrics?
* “JVM memory metrics” refer to the metrics exposed by the java virtual machine (JVM) that runs kafka brokers (and sometimes kafka clients). These metrics help you understand how kafka is using heap and non-heap memory, garbage collection behavior, and potential memory pressure.
* kafka itself doesn’t redefine these metrics — it exposes standard JVM metrics (usually via JMX) alongside kafka-specific ones.


## key kafka JVM memory metrics
### 1. heap memory usage
* these are the most important memory metrics for kafka brokers:
** jvm.memory.used
*** current amount of memory used (bytes)
*** usually broken down by heap / non-heap and memory pool

** jvm.memory.max
*** maximum memory available for a given pool (e.g., heap max from -Xmx)

** jvm.memory.committed
*** memory guaranteed by JVM for use (allocated but not necessarily used)

** jvm.memory.used / max ratio
*** often used to detect memory pressure

* typical heap pools:
** G1 eden space
** G1 survivor space
** G1 old gen (most important for kafka stability)


### 2. Non-heap memory metrics
* kafka also uses non-heap memory (class metadata, code cache, etc.)
** metaspace - jvm.memory.used{area="nonheap", pool="Metaspace"}
** code cache - stores JIT compiled code
* high metaspace usage can indicate classloader leaks or heavy plugin usage.


### 3. garbage collection (GC) metrics
* GC is tightly tied to memory health:
** jvm.gc.pause - time spent in GC pauses (very important for latency)
** jvm.gc.memory.promoted - memory moved from young to old generation
** jvm.gc.memory.allocated - total allocated memory over time
** GC count / time per collector - e.g., G1 young GC, G1 old GC
* long GC pauses in Kafka often cause:
** increased produce/consume latency
** request timeouts
** ISR instability


### 4. thread-related memory impact (indirect)
* not memory metrics directly, but relevant:
** jvm.threads.live
** jvm.threads.daemon
** more threads = more stack memory usage.


## where these metrics come from in kafka
* kafka exposes JVM metrics via:
** JMX (Java Management Extensions) ← most common
** metrics reporters:
*** prometheus JMX exporter
*** confluent control center (if using confluent platform)
** you typically see them under:
*** java.lang:type=Memory
*** java.lang:type=GarbageCollector
*** java.lang:type=MemoryPool


## why JVM memory metrics matter in kafka
* kafka is very sensitive to JVM memory because it:
** uses heap for request handling, metadata, and broker logic
** relies on OS page cache (not heap) for log storage
** is affected by GC pauses more than raw memory usage
* so you usually monitor:
** heap usage staying stable (no continuous growth)
** GC pause times staying low (often < 100–200ms ideal)
** No old-gen saturation


## common real-world alert thresholds
* heap usage > 70–80% sustained → warning
* old gen continuously growing → possible memory leak or backlog
* GC pause spikes > 1s → serious kafka latency risk
* metaspace steadily growing → classloader leak risk


## quick mental model
* kafka JVM memory health = heap stability + low GC pause time + stable old generation




# what is directBufferMemory metric?
* directBufferMemory refers to a metric that tracks off-heap (direct) memory used by the JVM for NIO direct byte buffers, mainly for network I/O operations.


## What it actually means
* kafka uses Java NIO (non-blocking I/O) for network communication. instead of allocating byte arrays on the Java heap, it often uses direct buffers, which are allocated outside the JVM heap in native memory.
* so directBufferMemory measures how much of that off-heap direct memory is currently in use or allocated by kafka (via the JVM).


## why kafka uses direct buffers
* direct buffers are used because they:
** avoid copying data between JVM heap and native OS buffers
** improve throughput for network-heavy workloads
** re more efficient for socket read/write operations


## where it shows up in kafka
* depending on how you're monitoring kafka, you might see it as:
** JVM metric: java.nio:type=BufferPool,name=direct
** attributes like:
*** MemoryUsed
*** TotalCapacity
*** Count
* in monitoring systems (Prometheus/JMX exporters), it may appear as:
** directBufferMemoryUsed
** directBufferCount


## what it represents in practice
* it reflects memory used by things like:
** network request/response buffers
** socket send/receive buffers
** internal Kafka network threads (e.g., NetworkClient, SocketServer)
* it is not part of heap memory, so:
** it won’t show up in typical heap GC stats
** it can still cause OutOfMemoryError: Direct buffer memory


## why it matters
* if directBufferMemory grows too high or is mismanaged
** kafka brokers can run out of native memory
** you may see errors like: java.lang.OutOfMemoryError: Direct buffer memory
** performance can degrade under heavy network load


## common causes of high usage
* too many open connections (producers/consumers)
* large message batches
* high network throughput without enough configured limits
* misconfigured socket.send.buffer.bytes / socket.receive.buffer.bytes


## related configuration
* JVM limit controlling it: -XX:MaxDirectMemorySize
* kafka-side network tuning:
** num.network.threads
** socket.send.buffer.bytes
** socket.receive.buffer.bytes
** replica.fetch.max.bytes


## quick intuition
* think of directBufferMemory as: “how much native (off-heap) RAM kafka is currently using to move bytes across the network efficiently.”




# what is “garbage collection (GC) time metrics”?
* “garbage collection (GC) time metrics” refer to JVM-level metrics exposed by the Kafka broker that measure how much time the Java Garbage Collector spends pausing application threads. These are not Kafka-specific business metrics, but they are critical for broker performance and latency.

## Here are the key GC-related metrics you’ll typically see:

### 1. JVM GC time (most important)
* kafka exposes GC time via standard JVM MBeans (via JMX):
** java.lang:type=GarbageCollector,name=G1 Young Generation
** java.lang:type=GarbageCollector,name=G1 Old Generation (names vary depending on GC algorithm: G1, CMS, ZGC, etc.)
* key attributes:
** CollectionTime → total time spent in GC (milliseconds)
** CollectionCount → number of GC events
* you typically compute:
** GC time rate = delta(CollectionTime) / time window
** avg pause per GC = delta(CollectionTime) / delta(CollectionCount)


### 2. kafka broker-level GC metrics (JMX-derived / monitoring tools)
* kafka itself doesn’t redefine GC metrics, but monitoring systems (prometheus, JMX exporters, confluent metrics) often expose:
** kafka.server:type=BrokerTopicMetrics (indirect impact via latency)
** kafka.network:type=RequestMetrics (latency spikes correlate with GC pauses)
* more importantly, you observe GC indirectly through:
** request latency spikes
** request handler idle time drops
** URP during long pauses


### 3. common GC performance indicators used in kafka monitoring
* these are the metrics operators actually watch:
** GC pause time (ms) - total stop-the-world time
** GC Pause Percent (%) - GC time / wall clock time
*** sustained >1–2% is often concerning for Kafka brokers.
** max GC pause - worst-case pause (important for latency-sensitive clusters)
** young vs old GC time
*** young GC: frequent, usually short
*** old GC: infrequent but can cause major outages if long


### 4. why GC time matters in kafka
* high GC time leads to:
** increased produce/consume latency
** request timeouts
** ISR shrink/expand instability
** controller election delays (in extreme cases)
* kafka brokers are very sensitive because they rely on:
** low-latency network threads
** fast request handler threads
** tight batching and memory buffering
* even short GC pauses (200–500 ms) can ripple into noticeable cluster instability.


### 5. practical monitoring setup (common in production)
* most teams track:
** jvm_gc_collection_seconds_sum (prometheus JVM exporter)
** jvm_gc_pause_seconds
** kafka_server_broker_topic_metrics_* latency metrics
** JVM heap usage + GC frequency together
* alerting example:
** GC pause > 500 ms sustained
** -or-
** GC time > 1% over 5 minutes window


### 6. if you're tuning kafka GC
* common JVM settings:
** G1GC (default in modern kafka)
** -XX:MaxGCPauseMillis=20-50 (typical target range)
** Heap sizing: avoid overcommitting (often 6–30 GB depending on broker role)




# what is NetworkProcessorAvgIdlePercent metric?
* The NetworkProcessorAvgIdlePercent metric in kafka measures how much time kafka’s network processor threads are idle (not doing work) on average.


## what it actually represents
* kafka brokers use a set of network processor threads to handle incoming/outgoing requests (producers, consumers, replication). this metric tells you: “out of total time, what fraction are those threads sitting idle?”
* value range: 0.0 to 1.0
* 1.0 (or ~100%) → threads are mostly idle (low load)
* 0.0 (or ~0%) → threads are fully busy (potential bottleneck)


## why it matters
* it’s a key signal of network-level saturation:
* high idle (e.g. 0.7–0.9): plenty of capacity, network threads are underutilized
* moderate (e.g. 0.3–0.6): healthy utilization depending on workload
* low (e.g. < 0.2): network threads are heavily loaded → requests may queue → latency increases
* near zero: strong indicator of a bottleneck at the broker’s network layer


## when to worry
* you should investigate if: the metric stays consistently low (< 0.2)
* you also see:
** increased request latency
** growing request queues (RequestQueueSize)
** throttling or timeouts
* common causes of low idle %
** too many client connections or requests
** large request sizes (big batches)
** insufficient num.network.threads
** CPU contention on the broker
** slow disk causing backpressure into network threads


## what you can do
* increase num.network.threads
* scale out brokers
* tune producer batch sizes / request rates
* check CPU and disk I/O bottlenecks
* reduce excessive connections (e.g., connection pooling)


## in short:
* NetworkProcessorAvgIdlePercent tells you how “busy vs free” Kafka’s network layer is — and low values are an early warning that your broker is struggling to keep up




# what is systemCpuLoad metric?
* the systemCpuLoad metric in kafka refers to how much CPU the entire system (host machine) is using — not just the kafka process itself.


## what it actually measures
* it represents the overall CPU utilization of the operating system
* typically expressed as a value between 0.0 and 1.0
** 0.0 → no CPU usage
** 1.0 → fully saturated CPU (100%)
* it includes all processes running on the machine, including kafka and everything else


## where it comes from
* kafka exposes this metric via JMX (java management extensions), and it’s usually backed by:
** the JVM’s OperatingSystemMXBean
** specifically something like: com.sun.management.OperatingSystemMXBean.getSystemCpuLoad()


## why it matters for kafka
* even though it’s not kafka-specific, it’s useful because:
** high systemCpuLoad can indicate resource contention
** kafka performance (throughput, latency) may degrade if:
*** CPU is saturated by Kafka itself
*** -or-
*** by other processes on the same host
* compare with related metric
** processCpuLoad → CPU used only by the kafka JVM process
** systemCpuLoad → CPU used by the entire machine
* example interpretation
** systemCpuLoad = 0.85
*** → 85% of total CPU capacity is in use
*** system is getting close to saturation




# what is processCpuLoad metric?
* the processCpuLoad metric in kafka shows how much CPU is being used by the kafka JVM process itself, not the whole machine.


## what it measures
* CPU utilization of just the kafka broker process
* reported as a value between 0.0 and 1.0
** 0.0 → kafka is idle
** 1.0 → kafka is fully using all available CPU cores


# where it comes from
* like systemCpuLoad, it’s exposed via JMX and typically backed by: com.sun.management.OperatingSystemMXBean.getProcessCpuLoad()


# how to interpret it
* processCpuLoad = 0.70
* → Kafka is using ~70% of the total CPU capacity available to it
* if your machine has multiple cores, this value reflects aggregate usage across all cores


# why it matters
* this metric tells you whether kafka itself is CPU-bound:
** high processCpuLoad → kafka is doing heavy work (e.g., high throughput, compression, replication)
** low processCpuLoad but high systemCpuLoad → something else on the machine is consuming CPU

* compare with systemCpuLoad
** processCpuLoad → kafka-only CPU usage
** systemCpuLoad → total CPU usage (kafka + everything else)

* practical insight
** high process + high system CPU → kafka is likely the main load
** low process + high system CPU → investigate other processes
** high process + low system CPU → kafka is busy, but system still has headroom