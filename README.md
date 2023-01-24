# Kafka Rest PoC
This is just a docker compose file to load Kafka and Kafka Rest Proxy to test the interaction.

## Local Test

1 - Start up Kafka and Kafka Rest:

```bash
docker compose up -d
```

2 - Need to export these two variables localted in `.envrc`:

```bash
export BOOTSTRAP_SERVERS='broker:29092'
export COMPOSE_IGNORE_ORPHANS=True
```

3 - Create a new topic, `users`, which we you can use to produce and consume events.
Use the kafka-topics command located inside the local running Kafka broker:

```bash
docker compose exec broker \
  kafka-topics --create \
    --topic users \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1
```

4 - Run the following command to produce some random user events to the users topic:

```bash
curl -X POST \                                                          
     -H "Content-Type: application/vnd.kafka.json.v2+json" \
     -H "Accept: application/vnd.kafka.v2+json" \                                 
     --data '{"records":[{"key":"1","value":"Srdjan"},{"key":"2","value":"Gustavo"},{"key":"3","value":"Dario"}]}' \
     "http://localhost:8082/topics/users"
```

You should see an output like this:

```bash
{
    "offsets":[
        {"partition":0,"offset":0,"error_code":null,"error":null},
        {"partition":0,"offset":1,"error_code":null,"error":null},
        {"partition":0,"offset":2,"error_code":null,"error":null}
    ],
    "key_schema_id":null,
    "value_schema_id":null
}  
```

5 - Create a consumer, starting at the beginning of the topic's log:

```bash
curl -X POST \
     -H "Content-Type: application/vnd.kafka.v2+json" \
     --data '{"name": "ci1", "format": "json", "auto.offset.reset": "earliest"}' \
     http://localhost:8082/consumers/cg1
```

6 - Subscribe to the topic users:

```bash
curl -X POST \
     -H "Content-Type: application/vnd.kafka.v2+json" \
     --data '{"topics":["users"]}' \
     http://localhost:8082/consumers/cg1/instances/ci1/subscription 
```

7 - Consume some data using the base URL in the first response: 

(Note that you must issue this command twice due https://github.com/confluentinc/kafka-rest/issues/432)

```bash
curl -X GET \
     -H "Accept: application/vnd.kafka.json.v2+json" \
     http://localhost:8082/consumers/cg1/instances/ci1/records 

sleep 10

curl -X GET \
     -H "Accept: application/vnd.kafka.json.v2+json" \
     http://localhost:8082/consumers/cg1/instances/ci1/records 
```

Verify that you see the following output returned from the REST Proxy:

```bash
[
    {"topic":"users","key":"jsmith","value":"alarm clock","partition":0,"offset":0},
    {"topic":"users","key":"htanaka","value":"batteries","partition":0,"offset":1},
    {"topic":"users","key":"awalther","value":"bookshelves","partition":0,"offset":2}
] 
```

8 - Close the consumer with a DELETE to make it leave the group and clean up its resources:

```bash
curl -X DELETE \
     -H "Content-Type: application/vnd.kafka.v2+json" \
     http://localhost:8082/consumers/cg1/instances/ci1 
```