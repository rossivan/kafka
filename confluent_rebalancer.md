confluent-rebalancer (confluent)
================================
* automatically generates redistribution plan based on stats i.e. automated intelligent balancing
* when the plan is executed, partitions are rebalanced (moved) across brokers in the cluster
* these are both manual steps to be done with this tool
* also called as auto data balancer
* Evenly distribute partition replicas
* optimization, the rebalancer tries to - evenly distribute partition replicas, balance disk usage, spread leaders across brokers, minimize movement cost
* it can generate heavy network + disk i/o
* not instantaneous (can take hours on larg clusters)
* should be run during low-traffic windows
* required confluent enterprise features


# dry run (ALWAYS do this), tells you
* proposed partition movements
* estimated data transfer
* balance improvements

confluent-rebalancer execute \
    --bootstrap-server localhost:9092 \
    --metrics-bootstrap-server localhost:9093 \
    --dry-run


# execute rebalance
confluent-rebalancer execute    \
    --bootstrap-server  kafka-1:9092    \
    --metrics-bootstrap-server  kafka-1:9092 \
    --throttle    1000000   \
    --verbose


# check status
confluent-rebalancer status \
    --bootstrap-server localhost:9092


# stop if needed
confluent-rebalancer stop \
    --bootstrap-server localhost:9092