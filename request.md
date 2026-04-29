# what is  RequestQueueSize and ResponseQueueSize metric?
* the metrics **RequestQueueSize** and **ResponseQueueSize** are broker-level indicators that help you understand how overloaded (or healthy) your network and request handling pipeline is.


## RequestQueueSize
* this measures the number of incoming client requests waiting in the broker’s request queue before being processed.
** think of it as a backlog of work.
** requests here include produce, fetch, metadata, etc.
** if this number grows:
*** the broker is receiving requests faster than it can process them.
*** possible causes:
**** CPU saturation
**** disk I/O bottlenecks
**** too few network or I/O threads
* a consistently high value is usually a warning sign of bottlenecks.


## ResponseQueueSize
* this measures the number of responses waiting to be sent back to clients.
** after kafka processes a request, the response goes into this queue.
** if this number grows:
*** the broker is struggling to send responses quickly enough.
*** possible causes:
**** network bandwidth limitations
**** slow clients consuming responses
**** insufficient network threads
* high values indicate **network/output pressure**, not processing delay.


## key difference (simple mental model)
* RequestQueueSize → “how backed up is incoming work?”**
* ResponseQueueSize → “how backed up is outgoing responses?”**


## how to interpret them together
* high RequestQueueSize, low ResponseQueueSize → processing bottleneck (CPU, disk, threads)
* low RequestQueueSize, high ResponseQueueSize → network/output bottleneck
* both high → system is overloaded end-to-end
* both low → healthy system (or low traffic)


## practical tip
* you’ll usually monitor these alongside:
** request latency
** network processor idle %
** I/O wait
** throughput (bytes in/out)
* map these metrics to specific kafka configs (like `num.network.threads`, `num.io.threads`)




# what is RequestHandlerAvgIdlePercent?
* RequestHandlerAvgIdlePercent is a key broker metric that tells you how busy (or overloaded) the request processing threads are.


## What it measures
* it represents the average fraction of time that kafka’s request handler threads are idle (i.e., not doing work).
* value range: 0.0 to 1.0
** 1.0 → completely idle (no load)
** 0.0 → fully busy (maxed out)


## these “request handler” threads come from the `num.io.threads` pool and are responsible for:
* processing produce/fetch requests
* handling replication
* doing disk I/O work


## how to interpret it
* high value (e.g., 0.7 – 1.0) → Threads are mostly idle → Broker has plenty of capacity
* moderate value (e.g., 0.3 – 0.6) → normal under steady load
* low value (e.g., < 0.2) → threads are busy most of the time → broker may be approaching saturation
* near 0.0 consistently → strong sign of overload or bottleneck → requests may start queuing (you’ll likely see RequestQueueSize increase)


## how it relates to other metrics
* low RequestHandlerAvgIdlePercent + high RequestQueueSize → clear processing bottleneck
* low idle percent + low queue size → system is busy but keeping up (borderline)
* high idle percent + high latency → problem is likely elsewhere (network, clients, disk)


## simple mental model
* think of it as: “how much free time do my worker threads have?”
* plenty of free time → system is relaxed
* no free time → system is stressed


## practical tuning insight
* if this metric is consistently low:
** you might increase `num.io.threads`
* -or-
* investigate:
** disk throughput
** CPU usage
** request patterns (large batches, slow disks, etc.)




# what is ProduceRequestsPerSec?
* ProduceRequestsPerSec is a broker-level metric that tells you how many produce requests the broker is handling per second.


## what it actually measures
* a produce request is when a producer client sends data (messages/records) to kafka. so:
* ProduceRequestsPerSec = number of incoming produce API requests per second
* it’s counted per broker, not per topic (unless you break it down further with tags in your monitoring system)


## Important nuance
* this metric tracks requests, not messages. one produce request can contain:
* a single message
* -or-
* a batch of thousands of messages. so:
* high ProduceRequestsPerSec ≠ high throughput (necessarily)
* low ProduceRequestsPerSec with large batches can still mean very high data throughput


## why it matters
* it helps you understand:
** load on brokers (more requests = more CPU/network overhead)
** producer behavior (batching vs sending one message at a time)
** efficiency of your pipeline


## how to interpret it
* high value + low throughput → producers likely not batching efficiently
* low value + high throughput → good batching, efficient usage
* sudden spikes → traffic bursts or producer misconfiguration
* drops to near zero → producers may be down or disconnected


## related metrics (often more meaningful together)
* BytesInPerSec → actual data throughput
* RequestLatencyMs → how long requests take
* MessagesInPerSec → number of records ingested


## quick example
* 1,000 requests/sec with 10 messages each → 10,000 messages/sec
* 100 requests/sec with 1,000 messages each → 100,000 messages/sec (more efficient)




# what is TotalProduceRequestsPerSec metric?
* TotalProduceRequestsPerSec is a broker metric that counts the total number of produce requests handled per second, aggregated across all request outcomes


## how it differs from ProduceRequestsPerSec
* the naming is a bit confusing, but the key idea is:
** TotalProduceRequestsPerSec → counts all produce requests, regardless of result
** it includes:
*** successful requests
*** failed requests (e.g., authorization errors, timeouts, invalid requests)
** depending on your metrics system (JMX/Prometheus), this is often exposed as:
*** kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Produce and may be labeled as “total” to distinguish it from breakdowns by response type


## why it matters
* this metric helps you see overall load hitting the broker, even if things are going wrong.
* for example:
** if TotalProduceRequestsPerSec is high but successful requests are dropping → something is failing (auth, disk, ISR issues, etc.)
** if both total and success are high → system is under heavy but healthy load

## how to interpret it
* think of it as: TotalProduceRequestsPerSec = success + errors + throttled requests (per second)
* so it’s useful when paired with:
** error-rate metrics (e.g., failed produce requests)
** request latency
** throttle time
** BytesInPerSec (to correlate load vs actual throughput)


## quick example
* TotalProduceRequestsPerSec = 5,000
* successful = 4,500
* failed = 500
* → you’ve got a 10% failure rate, which this metric alone wouldn’t show—but it tells you the true incoming pressure on the broker.




# what is ProduceTotalTimeMs, ProduceLocalTimeMs, ProduceRemoteTimeMs?
* these metrics come from producer performance tracking, and they break down how long a produce request takes at different stages.

## 1. ProduceTotalTimeMs
* this is the end-to-end ti for a produce request, measured in milliseconds.
* it includes:
** time spent in the producer client (serialization, batching, queuing)
** network latency (sending request + receiving response)
** broker processing time (leader + replication work)
* 👉 in short: “How long did it take from send() to acknowledgment?”


## 2. ProduceLocalTimeMs
* this is the time spent on the broker that receives the request (leader)
* it includes:
* time to validate the request
* writing the message to the leader’s local log
* initial processing before replication completes
👉 think: “How long did the leader broker itself spend handling this request?”

## 3. ProduceRemoteTimeMs
* this measures time spent waiting on replication to follower brokers
* it includes:
* time for replicas to fetch the data
* waiting for required acknowledgments (acks=all especially)
👉 think: “How long did we wait for other brokers to catch up?”


## how they relate
* roughly: ProduceTotalTimeMs ≈ ProduceLocalTimeMs + ProduceRemoteTimeMs + network/client overhead


## why this breakdown matters
* high ProduceLocalTimeMs → broker is slow (disk I/O, CPU, contention)
* high ProduceRemoteTimeMs → replication lag (slow followers, network issues, ISR shrinking)
* high ProduceTotalTimeMs but low others → likely client-side or network latency


## practical example
if you see:
* Total = 120 ms
* Local = 20 ms
* Remote = 80 ms
* → The bottleneck is replication, not the leader itself.




# what is ProduceRequestQueueTimeMs?
* ProduceRequestQueueTimeMs is a broker-side metric that tells you how long produce (write) requests sit in the request queue before the broker starts processing them.


## what it actually measures
* when a producer sends messages to kafka:
** the request arrives at the broker.
** it gets placed in a request queue (shared with other requests).
** a network or I/O thread eventually picks it up and processes it.
* 👉 ProduceRequestQueueTimeMs = time spent waiting in that queue (in milliseconds)


## why it matters
* this metric reflects broker load and request contention:
* low values (ideal): requests are picked up quickly → broker is keeping up.
* high values (problem): requests are piling up → broker is overloaded or constrained.


## common causes of high queue time
* if you see spikes or sustained high values, it often points to:
** too many incoming requests (high producer throughput)
** insufficient network threads (num.network.threads)
** CPU saturation on broker
** disk I/O bottlenecks (slow writes, flush delays)
** large batch sizes or request sizes
** contention with other request types (fetch, metadata, etc.)


## related metrics (use together)
* to diagnose properly, you usually look at this alongside:
** RequestQueueSize → how many requests are waiting
** ProduceRequestQueueTimeMs → how long they wait
** LocalTimeMs → processing time inside broker
** TotalTimeMs → end-to-end broker time
** RequestHandlerAvgIdlePercent → thread saturation

## quick intuition
* think of it like a checkout line:
** queue time = waiting in line
** processing time = cashier handling your items
* even if processing is fast, long lines (queue time) mean the system is overloaded.

## rule of thumb
* consistently low (near 0 – a few ms): healthy
* spikes under load: normal
* sustained high values (tens/hundreds of ms): needs investigation




# what is ProduceRequestBytes?
* ProduceRequestBytes is a broker metric that measures the size of incoming produce requests (in bytes).


## what it represents
* every time a producer sends data to kafka, it sends a produce request that contains one or more batches of records.
👉 ProduceRequestBytes = total number of bytes in those incoming requests
* this typically includes:
** message payloads (your actual data)
** record batch overhead
** protocol overhead (headers, metadata)


## how to interpret it
* kafka usually exposes this metric as a rate (bytes/sec) and sometimes as distribution stats (avg, max, etc.).
** high values: large volumes of data are being ingested by the broker
** low values: lower throughput or smaller messages


## why it matters
* this metric helps you understand ingestion throughput and load characteristics:
** capacity planning (network + disk usage)
** detecting unusually large messages or batches
** correlating with latency or queue time issues


## how it relates to other metrics
* you’ll get the most insight when you compare it with:
** ProduceRequestQueueTimeMs → are large requests causing queue delays?
** BytesInPerSec → overall inbound throughput
** RequestQueueSize → backlog buildup
** MessageSizeAvg / Max → size of individual records


## common patterns
* high bytes + low queue time: system is handling load well
* high bytes + high queue time: broker is struggling (CPU, network, disk)
* low bytes but high queue time: likely thread or configuration bottleneck


## simple mental model
* think of:
** ProduceRequestBytes → how much data is arriving
** ProduceRequestQueueTimeMs → how long it waits to be processed




# what is ProducePurgatorySize?
* ProducePurgatorySize is a broker metric that tells you how many produce requests are currently being held in the broker’s “purgatory” waiting for a condition to be satisfied before they can complete.


## what “purgatory” means here
* kafka uses the term purgatory for requests that can’t be completed immediately and must wait. for produce requests, this usually happens when:
** the producer is using acknowledgments like acks=all
** the broker is waiting for enough replicas (in-sync replicas, ISR) to confirm the write
** replication hasn’t caught up yet
* so instead of failing or responding early, Kafka parks the request temporarily in purgatory.


## what ProducePurgatorySize measures
* tt’s simply: the number of produce requests currently waiting in purgatory


## why it matters
* low / near zero → normal; requests are completing quickly
* spikes or consistently high values → something is slowing down replication or acknowledgments


## common causes of high values
* slow followers (replica lag)
* URPs
* network latency between brokers
* disk I/O bottlenecks on brokers
* too strict durability settings (acks=all with few ISR members)


## related metrics to check
* if ProducePurgatorySize is high, also look at:
** UnderReplicatedPartitions
** ISRExpandsPerSec / ISRShrinksPerSec
** RequestHandlerAvgIdlePercent
** ReplicaFetcherManager metrics (lag)


## quick intuition
* think of it like a queue:
* producers send messages
* kafka writes them
* if it needs to wait for replicas → the request goes into purgatory
* ProducePurgatorySize = how many are stuck waiting




# what is FetchConsumerRequestsPerSec?
* FetchConsumerRequestsPerSec measures how many fetch requests from consumers the broker is handling per second.


## hat it actually counts
* every kafka consumer periodically sends a fetch request to pull data from a broker. This metric tracks:
** the rate (requests/sec) at which the broker receives fetch requests from consumers
** it’s typically exposed as a meter (with 1-min, 5-min, 15-min rates).


## why it matters
* this metric gives you a sense of consumer activity and load on the broker:
** higher values → more frequent polling by consumers (or more consumers)
** lower values → fewer consumers, slower polling, or idle consumers


## how to interpret it
* healthy cluster: steady rate aligned with expected consumer throughput
* sudden spike:
** consumers restarted (they re-poll aggressively)
** misconfigured consumers (very low fetch.min.bytes or fetch.max.wait.ms)
** too many small fetches (inefficient consumption)
* drop to near zero:
** consumers disconnected or crashed
** network issues
** topic inactivity


## important nuance
* this metric counts requests, not data volume. so:
** many small fetches → high FetchConsumerRequestsPerSec
** few large fetches → lower value, but possibly higher throughput
* that’s why you should pair it with:
** BytesOutPerSec (actual data served)
** FetchConsumerTotalTimeMs (latency)
** MessagesInPerSec (producer side)


## quick intuition
* think of it as: “how often are consumers knocking on the broker’s door asking for data?”
* it doesn’t tell you how much they take—just how often they ask.




# what is NumIncrementalFetchSessions?
* the NumIncrementalFetchSessions metric refers to the number of currently active incremental fetch sessions on a broker.


## what that means in practice
* kafka uses something called incremental fetch sessions (introduced with KIP-227) to make consumer fetch requests more efficient. instead of sending the full list of partitions and metadata on every request, the client and broker maintain a session:
** the client sends a full fetch request once
** after that, it sends incremental updates (only changes)
** the broker caches session state to avoid redundant data transfer


## so the metric:
* NumIncrementalFetchSessions = count of active fetch sessions being tracked by the broker
* it increases when: consumers establish incremental fetch sessions
* it decreases when: sessions expire, consumers disconnect or stop fetching


## why it matters
* this metric helps you understand:
** broker memory usage: each session consumes memory (partition state, offsets, etc.)
** fetch efficiency: more incremental sessions → less network overhead per request
** consumer behavior patterns: a sudden drop might indicate consumer churn or instability


## related metrics (for context)
* you’ll often see it alongside:
** NumFetchSessions → total fetch sessions (incremental + non-incremental)
** NumIncrementalFetchPartitionsCached → partitions tracked in sessions
** IncrementalFetchSessionEvictionsPerSec → how often sessions are dropped


## when to worry
* too high → could mean memory pressure on brokers
* too low (unexpectedly) → clients may not be using incremental fetch (older clients or misconfig)




# what is IncrementalFetchSessionEvictionsPerSec?
* the IncrementalFetchSessionEvictionPerSec metric in kafka relates to how kafka brokers manage incremental fetch sessions for consumers.


## what it means
* kafka introduced incremental fetch sessions (via KIP-227) to optimize how consumers fetch data. Instead of sending full partition fetch requests every time, consumers maintain a session with the broker and only send changes (deltas).
* the metric IncrementalFetchSessionEvictionPerSec tracks:
** the rate (per second) at which incremental fetch sessions are being evicted (removed) from the broker’s cache.


## why sessions get evicted
* a fetch session may be evicted when:
** session cache is full → kafka has a bounded cache and removes older sessions (LRU-style)
** consumer inactivity → sessions expire if not used
** broker memory pressure
** client issues (e.g., disconnects, errors, or restarts)


## why this metric matters
* this metric is useful for diagnosing efficiency and stability of consumer fetch behavior:
** lLow / near zero: normal and expected. sessions are stable and reused effectively
** moderate but steady: could be normal under high churn (many short-lived consumers)
** high or spiky: indicates inefficiency:
*** vonsumers frequently losing sessions
*** increased network overhead (falling back to full fetch requests)
*** potential broker cache sizing issues


## what to check if it’s high and if you see elevated values:
* consumer behavior
** are consumers restarting frequently?
** are there many short-lived consumer groups?
* Broker configs
** max.incremental.fetch.session.cache.slots (session cache size)
** memory pressure on the broker
* network / stability
** connection churn or timeouts


## simple intuition
* think of incremental fetch sessions like a “conversation context” between consumer and broker:
** when reused → efficient, small updates
** when evicted → conversation resets → bigger, more expensive requests




# what is TotalFetchRequestsPerSec?
* the TotalFetchRequestsPerSec metric in kafka measures the rate of fetch requests received by a broker per second.


## what it represents
* kafka clients (both consumers and follower replicas) send fetch requests to brokers to read data.
* TotalFetchRequestsPerSec tells you how many of those requests the broker is handling each second.


## who generates fetch requests?
* consumers → pulling messages from topics
* follower brokers → replicating partitions from leaders
* so this metric reflects both consumption activity and replication traffic.


## how to interpret it
* high value:
** could mean strong consumer activity (lots of reads)
** -or-
** heavy replication (e.g., many partitions or lagging replicas)
* low value
** few consumers or slow polling
** possibly underutilized cluster


## why it matters
* helps you understand read load on brokers
* useful for diagnosing:
** consumer performance issues
** replication bottlenecks
* often analyzed alongside:
** BytesOutPerSec (actual data volume)
** FetchConsumerRequestsPerSec (consumer-only fetches)
** FetchFollowerRequestsPerSec (replication-only fetches)


## quick example: if:
* TotalFetchRequestsPerSec = 10,000
* but BytesOutPerSec is low
* → You might have inefficient consumers making frequent small fetches.




# what is NumIncrementalFetchPartitionsCached?
* the NumIncrementalFetchPartitionsCached metric in kafka relates to how kafka brokers handle incremental fetch sessions, which are an optimization used by consumers to reduce network and CPU overhead.


## what it measures
* NumIncrementalFetchPartitionsCached tracks the number of partitions currently stored (cached) in incremental fetch sessions on the broker.


## background (why this exists)
* kafka introduced incremental fetch requests (via fetch sessions) so consumers don’t have to resend the full list of partitions on every poll. instead:
** the consumer establishes a session with the broker
** he broker caches the partition list and state
** subsequent requests only send changes (add/remove partitions)
* so this metric tells you:
** how many topic-partitions are being tracked in those cached sessions
** essentially, the size of all active incremental fetch session caches


## why it matters
* higher values → more partitions cached → more memory usage on the broker
* very low or zero → incremental fetch may not be used effectively (clients might be using older protocol versions or sessions are constantly expiring/resetting)
* helps diagnose:
** consumer inefficiency
** roker memory pressure
** fetch session churn


## related metrics (for context)
* you’ll usually look at it alongside:
** NumIncrementalFetchSessions → how many sessions exist
** IncrementalFetchSessionEvictionsPerSec → how often sessions are dropped
** FetchSessionCacheHitsPerSec / MissesPerSec → efficiency of caching


## simple intuition
think of it like: “how many partitions are kafka brokers remembering for consumers so they don’t have to repeat themselves every time?”




# what is FetchConsumerTotalTimeMs and FetchConsumerLocalTimeMs?
* these two metrics come from kafka’s consumer-side performance tracking, and they’re mainly used to understand where time is being spent when a consumer fetches data.

## 🧩 FetchConsumerTotalTimeMs
* this is the total end-to-end time a consumer spends handling a fetch request.
* it typically includes:
** sending the fetch request to the broker
** network latency (round trip)
** broker processing time
** receiving the response
** plus any local processing time on the consumer side
* 👉 In short: TotalTime = Remote (broker + network) + Local (client-side work)


## 🧩 FetchConsumerLocalTimeMs
* this is the time spent locally on the consumer after the response is received.
* it includes things like:
** deserializing records
** decompressing message batches
** iterating through records
** internal bookkeeping in the consumer
* 👉 This is strictly client-side processing time, not network or broker time.


## 🧠 How to interpret them together
* these two metrics are most useful in comparison:
* if FetchConsumerTotalTimeMs is high but FetchConsumerLocalTimeMs is low → The bottleneck is likely network or broker-side latency
* if FetchConsumerLocalTimeMs is high → The consumer is spending significant time on:
** deserialization
** large batches
** CPU-heavy processing
* if both are high → you may have combined issues (slow broker + heavy client processing)

##⚡ Quick example
* totalTime = 120 ms
* localTime = 80 ms
* → ~40 ms is spent on network + broker
* → ~80 ms is spent inside your consumer


## 🛠️ Why these matter
* they help you:
** diagnose slow consumers
** tune batch sizes (fetch.max.bytes, max.poll.records)
** optimize deserialization or message formats
** separate infra problems vs application inefficiencies




# what is FetchFollowerRequestsPerSec?
* the FetchFollowerRequestsPerSec metric in kafka measures how frequently broker replicas (followers) send fetch requests to their leader broker to replicate data.


## here’s what that means in practice:
* what it tracks
** it counts the number of fetch requests per second made by follower replicas.
** these requests are how kafka keeps partitions replicated across brokers.


## Why it exists
* kafka uses a leader–follower model for each partition.
* followers continuously pull data from the leader (they don’t get pushed updates).
* this metric reflects how actively replication is happening.


## how to interpret it
* higher values → more replication traffic. this is normal in a busy cluster with many partitions or high write throughput.
* sudden drops → could indicate replication issues, follower lag, or broker/network problems.
* unusually high spikes → might signal frequent leader changes, follower catch-up after lag, or cluster instability.


## related context
* often analyzed alongside:
** FetchFollowerBytesPerSec (volume of data replicated)
** UnderReplicatedPartitions (health of replication)
* ReplicaFetcherManager.MaxLag (replication delay)


## in short
* FetchFollowerRequestsPerSec tells you how often followers are asking leaders for new data, making it a key signal for monitoring kafka replication activity and cluster health.




# what is FetchFollowerTotalTimeMs and FetchFollowerLocalTimeMs?
* FetchFollowerTotalTimeMs and FetchFollowerLocalTimeMs are broker-side request latency metrics related to how long the broker takes to handle follower replica fetch requests (i.e., replication traffic between brokers).
* they come from kafka’s request metrics system and are mainly used to monitor replication performance and broker load.


## 🔹 FetchFollowerTotalTimeMs
* this is the end-to-end time a broker spends handling a follower fetch request.
* it includes:
** time the request spends in the network layer (socket receive → response send)
** time waiting in request queues before being processed
** time executing the fetch logic
** serialization + response sending time
* 👉 Think of it as: “how long did it take from request arrival to response fully sent?”


## 🔹 FetchFollowerLocalTimeMs
* this measures only the time spent executing the request inside Kafka’s broker logic.
* it excludes:
** network transmission time
** time waiting in request queues
** socket send/receive overhead
* it mainly includes:
** fetching log data from disk/page cache
** processing replica fetch logic
** building the response
* 👉 Think of it as: “how long did kafka’s core replication logic take?”


## 📊 why this matters
* the gap between the two helps you diagnose bottlenecks:
** large gap (Total ≫ Local) → network delay, request queueing, or broker overload
** both high → slow disk I/O, heavy replication load, or CPU pressure


## 🧠 in practice
* these metrics are useful for:
** monitoring replication lag health
** detecting broker saturation
** debugging ISR instability or slow followers




# what is FetchPurgatorySize?
* FetchPurgatorySize is a broker-side metric that tells you how many fetch requests are currently waiting in “purgatory” to be satisfied.
* to understand it, you need the concept of request purgatory in kafka.


## what “purgatory” means in kafka
* on a kafka broker, some requests can’t be immediately completed because the required data isn’t ready yet. Instead of failing or blocking threads, Kafka puts them into a holding area called purgatory.
* for fetch requests specifically (used by consumers and replicas), a request may be delayed because:
** the requested data has not been produced yet (not enough data in the partition)
** the consumer is waiting for a minimum amount of data (fetch.min.bytes)
** the request is waiting for its timeout or long-poll interval (fetch.max.wait.ms)
** in some cases, replication fetches are waiting for new log entries


## so what is FetchPurgatorySize?
* FetchPurgatorySize = the number of FetchRequests currently waiting in this delayed queue on the broker.
* in other words:
** 0 → all fetch requests are being served immediately
** > 0 → some consumers or replicas are waiting for data or conditions to be met


# why it matters - this metric is useful for diagnosing consumer lag or broker bottlenecks:
* high or growing FetchPurgatorySize
** consumers are waiting for data
** partitions may be underproducing
** fetch settings may be too aggressive (e.g., large fetch.min.bytes)
** broker may be under load or slow to serve data
* consistently zero
** either traffic is low, or data is always available immediately


# related context
* purgatory exists for multiple request types, not just fetch:
** ProducePurgatory (for produce acknowledgments)
** FetchPurgatory (for consumer/replica fetches)
** delayed operations like topic deletion or leader election tasks


## simple mental model
* think of FetchPurgatory as: “a waiting room on the broker where fetch requests sit until kafka has enough data (or enough time has passed) to respond.”