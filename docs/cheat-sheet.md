## CLOUD Platform:

5GMETA Cloud platform has been deployed in AWS EKS service. In the following picture we can see the architecure and the diferent services exposed: ![Cloud Architecture](img/Cloud%20Architecture.png)

The next figure shows the modules that form this cloud platform: ![Cloud Platform](img/Cloud%20Platform.png)

### Cloud Services shortcuts


**APIs:**

- Discovery API: [https://<ip>/discovery-api/](https://<ip>/discovery-api/ui)
- Instance API: [https://<ip>/cloudinstance-api/](https://<ip>/cloudinstance-api/ui)
- Dataflow API: [https://<ip>/dataflow-api/](https://<ip>/dataflow-api/ui)
- License API: [https://<ip>/license-api/](https://<ip>/license-api/ui)


**Kafka Platform:**

- Brokers (Bootstrap): **<ip>:31090**, **<ip>:31091**, **<ip>:31092**
- Registry: **<ip>:31081**
- KSQLDB: **<ip>:31088**
- Kafka UI: [http://<ip>:31080](http://<ip>:31080/) ( **<user>** / **<password>** )

**Other services:**

- Keycloak: [https://<ip>/identity/admin/](https://<ip>/identity/admin/) ( <user> / <password> )
- Grafana: [http://<ip>:31137](http://<ip>:31137) ( <user> / <password> )
- Prometheus: [http://<ip>:30090](http://<ip>:30090)
- Kubernetes Dashboard: [https://<ip>:30183](https://<ip>:30183). To get a token run: ``kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"``
- MySQL: **<ip>:30057**. To get the root password run: ``kubectl get secret --namespace mysql mysql-cluster -o jsonpath="{.data.mysql-root-password}" | base64 -d``. Another user has been created so that root user is not used in the applications or if the K8s secret is not used: **<user>** / **<password>**

:raising_hand: All the urls pointing to the public IPs of amazon can be replaces with any other public IP reserved in AWS. For example: **<ip>**

## MEC Platform:

Architecture of the MEC Platform: ![MEC Architecture](img/MEC%20Architecture.png)

Modules composing the platform: ![MEC Platform](img/MEC%20Platform.png)

### MEC Services shortcuts

**APIs:**

- Registration: [http://<ip>:12346](http://<ip>:12346/ui)
- Instance API: [http://<ip>:5000](http://<ip>:5000/ui) (Internal), [http://<ip>:5000](http://<ip>:5000/ui) (External)

**Brokers:**

- Message-broker: [http://<ip>:8161](http://<ip>:8161) (UI), **<ip>:5673** (Port for producing data, clients), **<ip>:61616** (Port of ActiveMQ, Kafka Connectors, Internal), **<ip>:616161** (Port of ActiveMQ, Kafka Connectors, External)
- Video-broker: **<ip>:8443**,  **<ip>:<55000-55098>**

**Other services:**

- OSM: [http://<ip>](http://<ip>)
- OSM API: [https://<ip>:9999](https://<ip>:9999)
- Kubernetes Dashboard: [https://<ip>:27460](https://<ip>:27460). To get a token run: ``kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"``
- Grafana: [http://<ip>:3000](http://<ip>:3000) ( **<user>** / **<password>** )
- Prometheus: [http://<ip>:9090](http://<ip>:9090)
- MySQL: **<ip>:35430**. To get the root password run: ``kubectl get secret --namespace mysql mysql-cluster -o jsonpath="{.data.mysql-root-password}" | base64 -d``

## Repositories

- **DockerHub**: [https://hub.docker.com/orgs/5gmeta/repositories](https://hub.docker.com/orgs/5gmeta/repositories)
- **Helm Chart repository**: [https://github.com/5gmeta/helmcharts](https://github.com/5gmeta/helmcharts)
- **OSM descriptors repository**: [https://github.com/5gmeta/vnfdescriptors](https://github.com/5gmeta/vnfdescriptors)
