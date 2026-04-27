kafka-configs   \
    --bootstrap-server  kafka-1:9092    \
    --entity-type   brokers \
    --entity-name   5   \
    --describe


kafka-configs   \
    --bootstrap-server  kafka-1:9092    \
    --entity-type   topics  \
    --entity-name   my-topic    \
    --describe