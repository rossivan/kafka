kafka-topics    \
    --list  \
    --bootstrap-server  kafka-1:9092


kafka-topics    \
    --describe  \
    --topic topic-1 \
    --bootstrap-server  kafka-1:9092


kafka-topics    \
    --describe  \
    --bootstrap-server  kafka-1:9092    \
    --under-replicated-partitions