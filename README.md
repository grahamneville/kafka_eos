# kafka_eos

This shows a POC for using TerminAttr on vEOS to stream data from vEOS SysDB to Kafka.

All of this is done using Docker containers.

Some sources:
https://github.com/spotify/docker-kafka
https://github.com/grahamneville/vrnetlab
https://github.com/aristanetworks/goarista/tree/master/cmd/ockafka
https://github.com/burnyd/eos-docker-HAproxy/blob/master/scripts/leaf1a.sh
https://gist.github.com/abacaphiliac/f0553548f9c577214d16290c2e751071



First we need to start a Kafka container, for ease we are using a container built by Spotify that contains Kafka and Zookeeper in one image.


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


Second, we use vrnetlab to spin up a vEOS image. Here we are using 4.20.1F. Details on how to spin up vEOS are out of scope here but the modifications to the vrnetlab.sh script and DockerFile are captured here.

Edit common/vrnetlab.py to add the ability to listen to 6042 where gRPC is going to listen on the vEOS container:

Add/Alter these lines in the revelent places:
```
        res.append("user,id=p%(i)02d,net=10.0.0.0/24,tftp=/tftpboot,hostfwd=tcp::2022-10.0.0.15:22,hostfwd=udp::2161-10.0.0.15:161,hostfwd=tcp::2830-10.0.0.15:830,hostfwd=tcp::26042-10.0.0.15:6042" % { 'i': 0 })
        
        run_command(["socat", "TCP-LISTEN:6042,fork", "TCP:127.0.0.1:26042"], background=True)       
```

Edit veos/docker/Dockerfile and expose port 6042

```
EXPOSE 22 161/udp 830 5000 6042 10000-10099
```


Now, Once the vEOS container is up and running add the following config:

```
conf t

event-handler Terminattr
   trigger on-boot
   action bash /usr/bin/TerminAttr -grpcaddr 0.0.0.0:6042 -allowed_ips 0.0.0.0/0 -disableaaa
```



The final part is to use ockafka to connect to gRPC on vEOS and forward to Kafka

You will need to know the IP address of the vEOS container:

```
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-veos-router
```

Now pull down ockafka docker image and link to the kafka container from earlier.
-addrs is the vEOS container IP
-kafkatopic will be the topic name that was set earlier
-subscribe doesn't need to be set but can if you only want a particular set of information

```
docker pull aristanetworks/ockafka
docker run --link kafka aristanetworks/ockafka -addrs 192.168.0.24 -kafkaaddrs kafka:9092 -kafkatopic arista_test -subscribe /Sysdb/environment/
```

Once all up and running on the consumer terminal window you should start to see messages coming through:

```
{"dataset":"192.168.0.24","timestamp":1513806392934,"update":{"Sysdb":{"environment":{"thermostat":{"coolingDomain":{"name":"coolingDomain","ready":false}}}}}}
{"dataset":"192.168.0.24","timestamp":1513806392934,"update":{"Sysdb":{"environment":{"thermostat":{"config":{"fanSpeed":{"value":0},"mode":0,"name":"config","pollInterval":5,"shutdownOnInsufficientFans":true,"shutdownOnOverheat":true}}}}}}
{"dataset":"192.168.0.24","timestamp":1513806392934,"update":{"Sysdb":{"environment":{"thermostat":{"status":{"airflowDirection":0,"coolingAlarmLevel":0,"inletTemperature":{"value":"-Infinity"},"mode":0,"name":"status","numFanTrayFailures":0,"temperatureAlarmLevel":0}}}}}}
{"dataset":"192.168.0.24","timestamp":1513806392935,"update":{"Sysdb":{"environment":{"thermostat":{"hwconfig":{"controlLoop":0,"name":"hwconfig","psuFansIndependent":false,"targetTemperatureCorrectionEnabled":true,"thermalPolicy":0}}}}}}
{"dataset":"192.168.0.24","timestamp":1513806392935,"update":{"Sysdb":{"environment":{"cooling":{"config":{"name":"config"}}}}}}
{"dataset":"192.168.0.24","timestamp":1513806392935,"update":{"Sysdb":{"environment":{"temperature":{"config":{"name":"config"}}}}}}
{"dataset":"192.168.0.24","timestamp":1513806392935,"update":{"Sysdb":{"environment":{"power":{"config":{"name":"config"}}}}}}
```



