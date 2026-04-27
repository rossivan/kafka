kafka-leader-elections
======================
* leader election tool is used to manually trigger partition leader changes - for maintenance, broker failures, or rebalancing
* 2 main tools: kafka-preferred-replica-election (classic) and kafka-leader-election (more flexible)
* switches leaders only, not data
* if brokers are unevenly loaded, only rebalancing (confluent-rebalancer) fixes it


# kafka-preferred-replica-election
# this triggers leader elections for all partitions in the cluster
kafka-preferred-replica-election.sh \
    --bootstrap-server localhost:9092


# to target specific topics / partitions create a json file
election.json
=============
{
  "partitions": [
    {"topic": "orders", "partition": 0},
    {"topic": "orders", "partition": 1}
  ]
}

kafka-preferred-replica-election.sh \
    --bootstrap-server localhost:9092 \
    --path-to-json-file election.json


# kafka-leader-election

# preferred leader election
kafka-leader-election.sh \
    --bootstrap-server localhost:9092 \
    --election-type PREFERRED \
    --all-topic-partitions

# unclean leader election (⚠️ risky)
# ⚠️ This can cause data loss because it may pick an out-of-sync replica
kafka-leader-election.sh \
    --bootstrap-server localhost:9092 \
    --election-type UNCLEAN \
    --all-topic-partitions

# to target specific topics / partitions create a json file
election.json
=============
{
  "partitions": [
    {"topic": "payments", "partition": 0}
  ]
}

kafka-leader-election.sh \
    --bootstrap-server localhost:9092 \
    --election-type PREFERRED \
    --path-to-json-file election.json