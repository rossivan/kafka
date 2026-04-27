kafka-reassign-partitions
=========================
* relocates partitions based upon JSON configuration file
* provides granular control
* manual process to create JSON configuration file
* broker will move to new 
* manual way to balance the brokers when you scale out or scale in brokers i.e. manual partition movement


reassign-topic-1.json
=====================
{
    "version": 1,
    "partitions": [
        {"topic": "topic-1", "partition": 0, "replicas": [1, 2, 3]},
        {"topic": "topic-1", "partition": 1, "replicas": [2, 3, 1]},
        {"topic": "topic-1", "partition": 2, "replicas": [3, 1, 2]},
        {"topic": "topic-1", "partition": 3, "replicas": [1, 2, 3]},
        {"topic": "topic-1", "partition": 4, "replicas": [2, 3, 1]},
        {"topic": "topic-1", "partition": 5, "replicas": [3, 1, 2]},
    ]
}

kafka-reassign-partitions.sh    \
    --bootstrap-server  server-1:39094    \
    --reassignment-json-file    reassign-topic-1.json   \
    --execute