### Deployment

```bash
kubectl create namespace kafka
helm install --name ke -f values.yaml --namespace kafka .

#upgrade
helm upgrade -f values.yaml ke .

#others
helm install --set cp-schema-registry.enabled=false,cp-kafka-rest.enabled=false,cp-kafka-connect.enabled=false confluent/cp-helm-charts

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

# Describe a topic
kafka-configs --bootstrap-server ke-cp-kafka-headless:9092 --entity-type brokers --entity-default --describe
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name smap_telemetry_data --describe

# Add config
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name smap_telemetry_data --alter --add-config retention.ms=604800000
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name druid_telemetry_data --alter --add-config retention.ms=172800000

kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name test_smap_telemetry_data --alter --add-config retention.ms=172800000
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name dev_druid_telemetry_data --alter --add-config retention.ms=172800000

# Create a message
MESSAGE="`date -u`"

# Produce a test message to the topic
echo "$MESSAGE" | kafka-console-producer --broker-list ke-cp-kafka-headless:9092 --topic ke-topic

# Consume a test message from the topic
kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic smap_telemetry_data_test --max-messages 1

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic test_smap_telemetry_data
kafka-console-consumer --bootstrap-server ke-cp-kafka.kafka:9092 --topic test_smap_telemetry_data
kafka-console-consumer --bootstrap-server ke-cp-kafka-external-0:31090,ke-cp-kafka-external-1:31091,ke-cp-kafka-external-2:31092 --topic test_smap_telemetry_data

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic smap_telemetry_data

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic druid_telemetry_data --from-beginning

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --list

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --describe --group kafka_druid_republisher_group --members

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --describe --group kafka_druid_republisher_group --offsets


#testing from outside the cluster
docker run \
--rm \
confluentinc/cp-kafka:5.1.0 \
kafka-console-consumer --bootstrap-server kafka0.zenatix.com:31090,kafka1.zenatix.com:31091,kafka2.zenatix.com:31092 --topic smap_telemetry_data --from-beginning

docker run \
--rm \
confluentinc/cp-kafka:5.1.0 \
kafka-console-consumer --bootstrap-server 104.211.226.230:31090,104.211.201.77:31091,104.211.222.34:31092 --topic smap_telemetry_data --from-beginning
 
```

#### Add worker to mqtt connector
```bash
kubectl get pods -n kafka

kubectl exec -n kafka -it ke-cp-kafka-connect-5447f6bfb-d4kht -c cp-kafka-connect-server -- /bin/bash

#Prod connector
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "smap-mqtt-source-lenses", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO smap_telemetry_data SELECT * FROM telemetry/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Test connector
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "smap-mqtt-source-lenses-test", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO test_smap_telemetry_data SELECT * FROM /telemetry_test/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Update a connector

curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses-test
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses-test/config

# Updating a connector
curl -s -X PUT -H "Content-Type: application/json"  --data '{"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO smap_telemetry_data SELECT * FROM telemetry/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}' http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses/config

# Status of connector
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses/status
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses/tasks
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses/tasks/0/status

# Delete a connector
curl -X DELETE http://e-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses-test

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