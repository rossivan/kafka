delete_messages.json
====================
{
    "partitions": [
        {
            "topic": "topic-1",
            "partition": 0,
            "offset": <specify-offset>
        }
    ],
    "version": 1
}


kafka-delete-records    \
    --bootstrap-server=server-1:39094   \
    --offset-json-file  ~/kafka/delete_messages.json