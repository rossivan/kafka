# confluent self balancing clusters (SBC)
* more feature rich than confluent rebalancer aka confluent auto data balancer
* no additional tools to run (built into the brokers)
* cluster balance is continuously monitored so that rebalances run whenever they are needed
* much faster rebalancing (several orders of magnitude)
* while the plan is executing SBC throttles replication during rebalance to prevent negative impact to clients


## SBC runs on each broker
* collects metrics for that broker
* writes the metrics to an internal metrics collection topic

## SBC on controller node
* aggregates the metrics
* generates load balance plans based on goals
* executes the plan and exposes monitoring data to control center

## SBC auto rebalance trigger options
* added and removed brokers
* any uneven load

## uneven load condition is based upon
* disk usage and network usage
* number of partitions / replicas per broker
* leadership and rack awareness

