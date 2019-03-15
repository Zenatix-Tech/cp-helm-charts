### Deployment

```bash
kubectl create namespace kafka
helm install --name ke -f values.yaml --namespace kafka .

#upgrade
helm upgrade -f values.yaml ke .

```

### Testing containers
```bash
kubectl apply -f examples/kafka-client.yaml
kubectl apply -f examples/zookeeper-client.yaml

```

**Connection string for Confluent Kafka:**
```bash
ke-cp-zookeeper-0.ke-cp-zookeeper-headless:2181,ke-cp-zookeeper-1.ke-cp-zookeeper-headless:2181,ke-cp-zookeeper-2.ke-cp-zookeeper-headless:2181
```

#### Zookeeper
**Log into the Zookeeper Pod**
```bash
kubectl -n kafka exec -it zookeeper-client -- /bin/bash
```

**Use zookeeper-shell to connect in the zookeeper-client Pod:**
```bash
zookeeper-shell ke-cp-zookeeper:2181
```

**Explore with zookeeper commands, for example:**
```bash
# Gives the list of active brokers
ls /brokers/ids

# Gives the list of topics
ls /brokers/topics

# Gives more detailed information of the broker id '0'
get /brokers/ids/0
```

#### Kafka
**Log into the Pod**
```bash
kubectl -n kafka exec -it kafka-client -- /bin/bash
```

**Explore with kafka commands:**
```bash
# List all topics
kafka-topics --zookeeper ke-cp-zookeeper-headless:2181 --list

# Create the topic
kafka-topics --zookeeper ke-cp-zookeeper-headless:2181 --topic smap_telemetry_data --create --partitions 3 --replication-factor 1 --if-not-exists

# Create a message
MESSAGE="`date -u`"

# Produce a test message to the topic
echo "$MESSAGE" | kafka-console-producer --broker-list ke-cp-kafka-headless:9092 --topic ke-topic

# Consume a test message from the topic
kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic smap_telemetry_data_test --max-messages 1

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic test_smap_telemetry_data

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic smap_telemetry_data

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic druid_telemetry_data --from-beginning

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --list

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --describe --group kafka_druid_republisher_group --members

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --describe --group kafka_druid_republisher_group --offsets

```

#### Add worker to mqtt connector
```bash
kubectl get pods -n kafka

kubectl exec -n kafka -it ke-cp-kafka-connect-5447f6bfb-5lrbm -c cp-kafka-connect-server -- /bin/bash

#Prod connector
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "smap-mqtt-source-lenses", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.service.quality":"1", "connect.mqtt.kcql":"INSERT INTO smap_telemetry_data SELECT * FROM telemetry/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://localhost:8083/connectors

#Test connector
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "smap-mqtt-source-lenses-test", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.service.quality":"1", "connect.mqtt.kcql":"INSERT INTO test_smap_telemetry_data SELECT * FROM /telemetry_test/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://localhost:8083/connectors

curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/
curl -s -X GET http://localhost:8083/connectors/

curl -X DELETE http://localhost:8083/connectors/smap-mqtt-source-lenses-test

```

#### Monitoring
**Deploy kafka-manager**
```bash
kubectl apply -f yahoo-kafka-manager/
```

**Deploy burrow**
```bash
kubectl apply -f linkedin-burrow/
```

### Delete Everything
```bash
#Delete kafka manager
kubectl delete -f yahoo-kafka-manager/

#Delete linkedin burrow
kubectl delete -f linkedin-burrow/

#Delete test containers
kubectl delete -f examples/kafka-client.yaml
kubectl delete -f examples/zookeeper-client.yaml

#Delete helm charts
helm delete --purge ke

```