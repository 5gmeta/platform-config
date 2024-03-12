# Cluster monitoring

## Installing the Kubernetes Metrics Server
````
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get deployment metrics-server -n kube-system
kubectl get --raw /metrics
````

## Deploying Kubernetes Dashboard
````
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

kubectl apply -f eks-admin-service-account.yaml
````

To connect to the Kubernetes dashboard, get the token and expose the service:
````
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"

kubectl proxy
````

To access the dashboard endpoint, open the following link with a web browser:  [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#!/login) and paste the token got in the first command.

The service type of the dashboard can be switched to NodePort so that the dashboard is exposed permanently.

## Deploying Kube-Prometheus stack
The following helm installs the  [kube-prometheus stack](https://github.com/prometheus-operator/kube-prometheus), a collection of Kubernetes manifests,  [Grafana](http://grafana.com/)  dashboards, and  [Prometheus rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)  combined with documentation and scripts to provide easy to operate end-to-end Kubernetes cluster monitoring with  [Prometheus](https://prometheus.io/)  using the  [Prometheus Operator](https://github.com/prometheus-operator/prometheus-operator).

````
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install -n monitoring --create-namespace -f values.yaml prometheus-stack prometheus-community/kube-prometheus-stack --version 46.0
````

The values file used in the helm can be found in the following 5GMETA repository: [values.yaml](https://github.com/5gmeta/platform-config/blob/main/src/kube-prometheus-stack/values.yaml)

````
prometheus:
  service:
    type: NodePort
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
    retention: 30d
  enabled: true
  prometheusSpec:
    additionalScrapeConfigs: |
      - job_name: kafka
        static_configs:
          - targets:
            - kafkacluster-cp-kafka-connect.kafka.svc.cluster.local:5556
            - kafkacluster-cp-kafka-rest.kafka.svc.cluster.local:5556
            - kafkacluster-cp-ksql-server.kafka.svc.cluster.local:5556
            - kafkacluster-cp-schema-registry.kafka.svc.cluster.local:5556
            - kafkacluster-cp-zookeeper.kafka.svc.cluster.local:5556
            - kafkacluster-cp-kafka.kafka.svc.cluster.local:5556

alertmanager:
  service:
    type: NodePort
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi

grafana:
  service:
    type: NodePort
  persistence:
    enabled: true

````

To expose temporally the ports of Prometheus and Grafana services:
````
PROMETHEUS=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].metadata.name}')
GRAFANA=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n monitoring $PROMETHEUS 9090 &
kubectl port-forward -n monitoring $GRAFANA 3000 &
````
If you want to expose them constantly, change the service type to NodePort.
