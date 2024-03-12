# Kafka Infraestructure

## Infraestructure deployment

For deploying Kafka Platform, the solution from Confluent has been used: 
[https://github.com/confluentinc/cp-helm-charts](https://github.com/confluentinc/cp-helm-charts)

Several changes have been made in the helm chart provided by Confluent. For example, [Kafka UI](https://github.com/provectus/kafka-ui) has been added to the deployment files, so that is is deployed together with the rest of components. A service monitor for monitoring kafka has been also added.

The modified helm chart can be found in the 5GMETA's repositories: [https://github.com/5gmeta/stream-data-gateway/tree/main/src/prod-version](https://github.com/5gmeta/stream-data-gateway/tree/main/src/prod-version)

To install Kafka Platform helm chart, first clone the previous repository and go to [prod-version](https://github.com/5gmeta/stream-data-gateway/tree/main/src/prod-version) folder. Then install it:
````
git clone https://github.com/5gmeta/stream-data-gateway.git
cd stream-data-gateway/src/prod-version
helm install kafkacluster ./cp-helm-charts -n kafka
````

If for some reason the official (no modified )helm has to be installed:
````
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
helm repo update
helm install confluentinc/cp-helm-charts --name kafka --set cp-control-center.enabled=false
````

**Confluent Kafka Grafana dashboard:**
To be imported in Grafana, [https://github.com/5gmeta/stream-data-gateway/blob/main/src/prod-version/cp-helm-charts/5GMETA/confluent-open-source-grafana-dashboard.json](https://github.com/5gmeta/stream-data-gateway/blob/main/src/prod-version/cp-helm-charts/5GMETA/confluent-open-source-grafana-dashboard.json)

**Kafka-UI manifest for deploying the UI:**
It is deployed through the helm, the followinf manifest is the one that is being deployed:
````
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
  labels:
    app.kubernetes.io/component: kafka-ui
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: kafka-ui
  template:
    metadata:
      labels:
        app.kubernetes.io/component: kafka-ui
    spec:
      containers:
      - name: kafka-ui
        image: provectuslabs/kafka-ui:latest
        imagePullPolicy: Always
        ports:
          - name: http
            containerPort: 8080
            protocol: TCP
        env:
          - name: KAFKA_CLUSTERS_0_NAME
            value: 5gmeta-cloud
          - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
            value: kafkacluster-cp-kafka.kafka.svc.cluster.local:9092
#          - name: KAFKA_CLUSTERS_0_ZOOKEEPER
#            value: kafkacluster-cp-zookeeper.kafka.svc.cluster.local:2181
          - name: KAFKA_CLUSTERS_0_SCHEMAREGISTRY
            value: http://kafkacluster-cp-schema-registry.kafka.svc.cluster.local:8081
          - name: KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME
            value: kafka-connect
          - name: KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS
            value: http://kafkacluster-cp-kafka-connect.kafka.svc.cluster.local:8083
          - name: KAFKA_CLUSTERS_0_KSQLDBSERVER
            value: http://kafkacluster-cp-ksql-server.kafka.svc.cluster.local:8088
          - name: AUTH_TYPE
            value: "LOGIN_FORM"
          - name: SPRING_SECURITY_USER_NAME
            value: <user>
          - name: SPRING_SECURITY_USER_PASSWORD
            value: <password>
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
  labels:
    app.kubernetes.io/component: kafka-ui
  namespace: kafka
spec:
  type: NodePort
  ports:
    - port: 8080
      protocol: TCP
      targetPort: http
      nodePort: 31080
  selector:
    app.kubernetes.io/component: kafka-ui
````

## KSQL DB

For deploying a KSQL DB Client and run SQL commands, the following command can be used:
````
kubectl run tmp-ksql-cli -n kafka --rm -i --tty --image confluentinc/cp-ksqldb-cli:7.1.0 [http://kafkacluster-cp-ksql-server:8088](http://kafkacluster-cp-ksql-server:8088)
````
The commands can be also run though Kafka UI.

## Kafka Connect

Connectors configuration code base:
**Upload Connector:**
````
{
	"connector.class": "com.datamountaineer.streamreactor.connect.jms.source.JMSSourceConnector",
	"connect.jms.initial.context.factory": "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
	"tasks.max": 1,
	"connect.jms.url": "tcp://<IP>:<PORT>",
	"connect.jms.username": "<user>",
	"connect.jms.connection.factory": "ConnectionFactory",
	"connect.jms.queues": "jms-queue",
	"connect.jms.password": "<password>",
	"connect.progress.enabled": "true",
	"connect.jms.kcql": "INSERT INTO "<datatype>-<instance_type>" SELECT * FROM "<datatype>-<instance_type>" WITHTYPE TOPIC ",
	"name": "<CONNECTOR NAME>"
}
````
**Download Connector:**
````
{
	"connector.class": "com.datamountaineer.streamreactor.connect.jms.sink.JMSSinkConnector",
	"connect.jms.initial.context.factory": "org.apache.activemq.jndi.ActiveMQInitialContextFactory",
	"tasks.max": 1,
	"connect.jms.url": "tcp://<IP>:<PORT>",
	"connect.jms.username": "<user>",
	"connect.jms.connection.factory": "ConnectionFactory",
	"connect.jms.queues": "jms-queue",
	"connect.jms.password": "<password>",
	"connect.progress.enabled": "true",
	"connect.jms.kcql": "INSERT INTO event SELECT * FROM event WITHTYPE TOPIC WITHFORMAT JSON ",
	"topics": "event",
	"name": "<CONNECTOR NAME>"
}
````

## Schema Registry

Schema for **< datatype>-<instance_type>** topics. The name of the schema must be **< datatype>-<instance_type>-value**.
````
{
	"type": "record",
	"name": "jms",
	"namespace": "com.datamountaineer.streamreactor.connect",
	"fields": [
		{
			"name": "message_timestamp",
			"type": [
				"null",
				"long"
			],
			"default": null
		},
		{
			"name": "correlation_id",
			"type": [
				"null",
				"string"
			],
			"default": null
		},
		{
			"name": "redelivered",
			"type": [
				"null",
				"boolean"
			],
			"default": null
		},
		{
			"name": "reply_to",
			"type": [
				"null",
				"string"
			],
			"default": null
		},
		{
			"name": "destination",
			"type": [
				"null",
				"string"
			],
			"default": null
		},
		{
			"name": "message_id",
			"type": [
				"null",
				"string"
			],
			"default": null
		},
		{
			"name": "mode",
			"type": [
				"null",
				"int"
			],
			"default": null
		},
		{
			"name": "type",
			"type": [
				"null",
				"string"
			],
			"default": null
		},
		{
			"name": "priority",
			"type": [
				"null",
				"int"
			],
			"default": null
		},
		{
			"name": "bytes_payload",
			"type": [
				"null",
				"bytes"
			],
			"default": null
		},
		{
			"name": "properties",
			"type": [
				"null",
				{
					"type": "map",
					"values": [
						"null",
						"string"
					]
				}
			],
			"default": null
		}
	],
	"connect.name": "com.datamountaineer.streamreactor.connect.jms"
}
````

