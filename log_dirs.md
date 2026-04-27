# what are offline log directories?
* offline log directories refer to storage directories (on disk) that Kafka brokers can no longer use because they’ve encountered a failure or are otherwise inaccessible.


# What are “log directories”?
* kafka stores topic data (messages) as logs on disk
* each broker has one or more log directories (configured via log.dirs) where these partitions live.


# When does a log directory go “offline”?
* A log directory is marked offline when kafka detects a serious issue, such as:
** disk failure or corruption
** I/O errors
** filesystem becoming read-only
** hardware problems or sudden unavailability
** when this happens, kafka stops using that directory to avoid further data damage.


# what happens after that?
* partitions stored in that directory may become unavailable on that broker
* if replication is configured properly, other brokers can still serve the data
* kafka marks the directory as offline and continues operating with remaining healthy directories


# why it matters
* offline log directories are important because they:
** signal hardware or disk health problems
** can lead to URP
** may impact availability or durability if replication is insufficient 


# how you detect them
* you’ll typically see:
** broker logs mentioning “offline log directory”
** metrics like OfflineLogDirectoryCount > 0
** admin tools reporting affected partitions


# How to fix
* common actions include:
** repair or replace the faulty disk
** remove the directory from log.dirs if permanently bad
** restart the broker after fixing the issue
** rebalance partitions if needed


# the kafka-log-dirs command is one of the main ways you inspect log directories on brokers,
# including whether any of them are offline what the tool does
* kafka-log-dirs lets you query brokers and see:
** which log directories exist on each broker
** what partitions are stored in each directory
** the size of those partitions
** whether a directory is marked as offline

kafka-log-dirs  \
    --bootstrap-server localhost:9092   \
    --describe

# look for 'error' values
{
  "brokers": [
    {
      "broker": 1,
      "logDirs": [
        {
          "logDir": "/data1/kafka",
          "error": null,
          "partitions": [
            {
              "partition": "orders-0",
              "size": 104857600,
              "offsetLag": 0,
              "isFuture": false
            }
          ]
        },
        {
          "logDir": "/data2/kafka",
          "error": "org.apache.kafka.common.errors.KafkaStorageException",
          "partitions": []
        }
      ]
    }
  ]
}

kafka-log-dirs  \
    --bootstrap-server localhost:9092   \
    --describe  \
    --broker-list 3