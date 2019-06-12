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
kafka-topics --zookeeper ke-cp-zookeeper-headless:2181 --topic telemetry_data --create --partitions 3 --replication-factor 1 --if-not-exists

# Describe a topic
kafka-configs --bootstrap-server ke-cp-kafka-headless:9092 --entity-type brokers --entity-default --describe
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name telemetry_data --describe

# Add config
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name telemetry_data --alter --add-config retention.ms=604800000
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name druid_telemetry_data --alter --add-config retention.ms=172800000

kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name test_telemetry_data --alter --add-config retention.ms=172800000
kafka-configs --zookeeper ke-cp-zookeeper-headless:2181 --entity-type topics --entity-name dev_druid_telemetry_data --alter --add-config retention.ms=172800000

# Create a message
MESSAGE="`date -u`"

# Produce a test message to the topic
echo "$MESSAGE" | kafka-console-producer --broker-list ke-cp-kafka-headless:9092 --topic ke-topic

# Consume a test message from the topic
kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic bench_data --max-messages 1

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic test_telemetry_data
kafka-console-consumer --bootstrap-server ke-cp-kafka.kafka:9092 --topic test_telemetry_data
kafka-console-consumer --bootstrap-server ke-cp-kafka-external-0:31090,ke-cp-kafka-external-1:31091,ke-cp-kafka-external-2:31092 --topic test_telemetry_data

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic telemetry_data

kafka-console-consumer --bootstrap-server ke-cp-kafka-headless:9092 --topic druid_telemetry_data --from-beginning

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --list

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --describe --group kafka_druid_republisher_group --members

kafka-consumer-groups --bootstrap-server ke-cp-kafka-headless:9092 --describe --group kafka_druid_republisher_group --offsets


#testing from outside the cluster
docker run \
--rm \
confluentinc/cp-kafka:5.2.0 \
kafka-console-consumer --bootstrap-server kafka0.zenatix.com:31090,kafka1.zenatix.com:31091,kafka2.zenatix.com:31092 --topic smap_samhi

docker run \
--rm \
confluentinc/cp-kafka:5.2.0 \
kafka-console-consumer --bootstrap-server 104.211.226.230:31090,104.211.201.77:31091,104.211.222.34:31092 --topic telemetry_data --from-beginning
 
```

#### Add worker to mqtt connector
```bash
kubectl get pods -n kafka

kubectl exec -n kafka -it ke-cp-kafka-connect-6ffd957b8f-pmpx5 -c cp-kafka-connect-server -- /bin/bash

#Prod connector for subscribing telemetry/+/+/+ and publishing to telemetry_data
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "telmetry-telemetry_data", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"telmetry-telemetry_data-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO telemetry_data SELECT * FROM telemetry/+/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/iiitdarchiver/+/+ and publishing to smap_iiitdarchiver
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "iiitdarchiver-smap_iiitd", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"iiitdarchiver-smap_iiitd-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_iiitdarchiver SELECT * FROM telemetry/iiitdarchiver/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/dominos/+/+ and publishing to smap_dominos
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "dominos-smap_dominos", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"dominos-smap_dominos-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_dominos SELECT * FROM telemetry/dominos/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/trent/+/+ and publishing to smap_trent
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "trent-smap_trent", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"trent-smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_trent SELECT * FROM telemetry/trent/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/spar/+/+ and publishing to smap_spar
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "spar-smap_spar", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"spar-smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_spar SELECT * FROM telemetry/spar/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/motherdairy/+/+ and publishing to smap_motherdairy
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "motherdairy-smap_motherdairy", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"motherdairy-smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_motherdairy SELECT * FROM telemetry/motherdairy/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/iocl/+/+ and publishing to smap_iocl
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "iocl-smap_iocl", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"iocl-smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_iocl SELECT * FROM telemetry/iocl/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/samhi/+/+ and publishing to smap_samhi
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "samhi-smap_samhi", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"samhi-smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_samhi SELECT * FROM telemetry/samhi/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Prod connector for subscribing telemetry/rockman/+/+ and publishing to smap_rockman
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "rockman-smap_rockman", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"rockman-smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO smap_rockman SELECT * FROM telemetry/rockman/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Test connector for subscribing telemetry/test_archiver/+/+ and publishing to test_smap
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "test_archiver-test_smap", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"test_archiver-test_smap-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"true", "connect.mqtt.kcql":"INSERT INTO test_smap SELECT * FROM telemetry/test_archiver/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Stress Tester connector emqx benchmark
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "bench-test", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"bench-test-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO bench_data SELECT * FROM bench/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Test connector for locust topic
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "mqtt-source-lenses-test", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"mqtt-source-lenses-test-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO test_telemetry_data SELECT * FROM /telemetry_test/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

#Test connector for new vernemq topic
curl -s -X POST -H "Content-Type: application/json" --data '{"name": "dominos-test", "config": {"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://ve-vernemq.testing:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"dominos-test-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO smap_dominos SELECT * FROM telemetry/dominos/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors

# get a connector status
kubectl -n kafka exec -it kafka-client -- bin/bash
kubectl -n kafka exec -it kafka-client -- curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/

kubectl -n kafka exec -it ke-cp-kafka-connect-6ffd957b8f-j6p2k -c cp-kafka-connect-server -- curl -s -X GET http://localhost:8083/connectors/
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/mqtt-source-lenses
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/mqtt-source-lenses-test
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/mqtt-source-lenses-test/config

curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/trent-smap_trent/tasks/0/status
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/mqtt-source-lenses/tasks
curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/mqtt-source-lenses/tasks/0/status

curl -s -X GET http://ke-cp-kafka-connect.kafka:8083/connectors/mqtt-source-lenses-test/status


# Updating a connector
kubectl -n kafka exec -it kafka-client -- curl -s -X PUT -H "Content-Type: application/json" --data '{"connector.class": "com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector", "tasks.max":"1", "connect.mqtt.hosts":"tcp://mqtt.vernemq:1883", "connect.mqtt.username":"zenatix_mqtt_client", "connect.mqtt.password":"xitanez123", "connect.mqtt.client.id":"dominos-smap_dominos-client-id", "connect.mqtt.service.quality":"1", "connect.mqtt.clean":"false", "connect.mqtt.kcql":"INSERT INTO smap_dominos SELECT * FROM telemetry/dominos/+/+ WITHCONVERTER=`com.datamountaineer.streamreactor.connect.converters.source.BytesConverter`"}}' http://ke-cp-kafka-connect.kafka:8083/connectors/dominos-smap_dominos/config

#restart connector
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/trent-smap_trent/tasks/0/restart
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/iiitdarchiver-smap_iiitd/tasks/0/restart
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/dominos-smap_dominos/tasks/0/restart
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/spar-smap_spar/tasks/0/restart
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/motherdairy-smap_motherdairy/tasks/0/restart
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/iocl-smap_iocl/tasks/0/restart
curl -s -X POST http://ke-cp-kafka-connect.kafka:8083/connectors/test_archiver-test_smap/tasks/0/restart

# Delete a connector

curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/iocl-smap_iocl
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/dominos-smap_dominos
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/test_archiver-test_smap
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/motherdairy-smap_motherdairy
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/iiitdarchiver-smap_iiitd
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/trent-smap_trent
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/spar-smap_spar
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/dominos-test
curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/samhi-smap_samhi
kubectl -n kafka exec -it kafka-client -- curl -X DELETE http://ke-cp-kafka-connect.kafka:8083/connectors/smap-mqtt-source-lenses

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