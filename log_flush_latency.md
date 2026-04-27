# what is log flush latency?
* log flush latency refers to how long it takes for messages that have already been written to Kafka’s in-memory buffers (or OS page cache) to be persisted to disk (i.e., “flushed” so they’re durable)
* When a producer sends a message to kafka:
** kafka appends it to a log segment file (sequential disk write).
** that write usually goes to the OS page cache, not immediately to physical disk.
** the actual disk write (flush) happens later—either: periodically, or when explicitly forced.
** log flush latency = time between when kafka appends the messages to a log segment and when the data is actually on disk.


# why it matters
* durability: until data is flushed, it could be lost if the broker crashes.
* performance: immediate flushes reduce latency but hurt throughput.
* trade-off: kafka intentionally delays flushes to stay fast


# kafka settings related to it
* two main configs control flushing behavior:
** log.flush.interval.messages (flush after N messages)
** log.flush.interval.ms (flush after a time interval)