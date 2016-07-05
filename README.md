uMirrorMaker provides the ability to replicate between Kafka clusters in other data centers. With uMirrorMaker, you can publish to multiple regional Kafka clusters and aggregate in an aggregation Kafka cluster instead of publishing to a single Kafka cluster.
The current MirrorMaker design uses the Kafka high level consumer to consume data from multiple regional kafka clusters and produces to an aggregation kafka cluster. With this design, whenever a regional kafka broker is having an issue, it will cause rebalancing to happen across all of the active topics.

uMirrorMaker introduces a helix controller as the MirrorMaker Controller and uses a simple consumer in MirrorMaker Worker. This design achieves the following goals:

1. Stable MM: Rebalancing only during startup (when a node is added/ deleted)
2. Simple operations: easy to scale up cluster, no server restart for whitelisting topics
3. High throughput: max offset lag is consistently 0.
4. Time SLA (~5min)

# uMirrorMaker Quick Start

## Get the Code
Check out the uMirrorMaker project:
```
git clone git@github.com:uber/umirrormaker.git
cd umirrormaker
```
This project contains everything (both mirrormaker-controller and mirrormaker-worker) you’ll need to run uMirrorMaker.

## Build uMirrorMaker
Before you can run the uMirrorMaker, you need to build a package for it. This package is what your deployment tool uses to deploy uMirrorMaker.
```
mvn clean package
```
Or command below (the previous one will take a long time to run)
```
mvn clean package -DskipTests
```

## Set Up Local Test Environment
To test uMirrorMaker locally you will need two systems: [Kafka](http://kafka.apache.org/), and [ZooKeeper](http://zookeeper.apache.org/). The script “grid” is to help you setup these systems. Start by running:
- Modify permission for the scripts generated by Maven.
```
chmod u+x bin/pkg/*.sh
```
- The command below will download, install, and start ZooKeeper and Kafka (will start two Kafka systems: kafka1, which we use as source Kafka cluster, and kafka2, which we use as destination Kafka cluster).
```
bin/grid bootstrap
```
- Create a dummyTopic in kafka1 and produce some dummy data
```
./bin/produce-data-to-kafka-topic-dummyTopic.sh
```
- Check if the data is successfully produced to kafka1, you should open another console tab and execute the command below
```
./deploy/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181/cluster1 --topic dummyTopic
```
- You should get data as below:
```
Kafka topic dummy topic data 1
Kafka topic dummy topic data 2
Kafka topic dummy topic data 3
Kafka topic dummy topic data 4
…
```

## Start uMirrorMaker

**Example1**: Copy data from source Kafka to destination Kafka cluster

- Start uMirrorMaker Controller (you should keep it running)
```
./bin/pkg/start-controller-example1.sh
```

- Start uMirrorMaker Worker (you should keep it running, and it’s normal if you see kafka.consumer.ConsumerTimeoutException at this moment since no topic has been added for copying)
```
./bin/pkg/start-worker-example1.sh
```

- Add topic to uMirrorMaker Controller to start copying from kafka1 to kafka2
```
curl -X POST -d '{"topic":"dummyTopic", "numPartitions":"1"}' http://localhost:9000/topics
```

- to check if the data is successfully copied to kafka2, you should open another console tab and execute the command below
```
./deploy/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181/cluster2 --topic dummyTopic1
```

- And you will see the same messages produced in kafka1 as below:
```
Kafka topic dummy topic data 1
Kafka topic dummy topic data 2
Kafka topic dummy topic data 3
Kafka topic dummy topic data 4
…
```

**Example2**: Copy data from source Kafka to destination Kafka cluster without explicitly whitelisting topics

- Start uMirrorMaker Controller (you should keep it running)
```
./bin/pkg/start-controller-example2.sh
```

- Start uMirrorMaker Worker (you should keep it running, and it’s normal if you see kafka.consumer.ConsumerTimeoutException at this moment since no topic has been added for copying)
```
./bin/pkg/start-worker-example2.sh
```

- Create topic in kafka2; Example2 enables topic auto-whitelisting (if a topic is in both source and destination kafka clusters, mirror maker controller will auto whitelist this topic and start copying data), don’t need to whitelist topic manually
```
./deploy/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181/cluster2 --topic dummyTopic --partition 1 --replication-factor 1
```

- to check if the data is successfully copied to kafka2, you should open another console tab and execute the command below (You might need to wait for about 20 seconds for controller to refresh)
```
./deploy/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181/cluster2 --topic dummyTopic
```

- And you will see the same messages produced in kafka1 as below:
```
Kafka topic dummy topic data 1
Kafka topic dummy topic data 2
Kafka topic dummy topic data 3
Kafka topic dummy topic data 4
…
```

## Shutdown
After you’re done, you can clean everything up using the same grid script.
```
./bin/pkg/stop-all.sh
```

Congratulations! You’ve now setup a local grid that includes Kafka, and ZooKeeper, and run a uMirrorMaker on it.
