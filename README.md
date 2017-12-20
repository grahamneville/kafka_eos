# kafka_eos



Run container:
```
docker pull spotify/kafka
docker run -d -p 2181:2181 -p 9092:9092 --hostname kafka --env ADVERTISED_HOST=kafka --env ADVERTISED_PORT=9092 --name kafka spotify/kafka
```


Create Topic:
```
docker exec kafka /opt/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic arista_test
```

List Topics:
```
docker exec kafka /opt/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --list --zookeeper localhost:2181
```

In a new SSH window - start a producer
```
docker run -it --rm --link kafka spotify/kafka /opt/kafka_2.11-0.10.1.0/bin/kafka-console-producer.sh --broker-list kafka:9092 --topic test
```

In a new SSH window - start a consumer
```
docker run -it --rm --link kafka spotify/kafka /opt/kafka_2.11-0.10.1.0/bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic test --from-beginning
```

Now send messages on the producer window with each message ending in a carridge return, these should appear in the consumer window
