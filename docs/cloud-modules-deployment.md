# 5GMETA Modules Deployment

In the Cloud Platform we have the following modules:

- Apisix (Gateway)
- Keycloak (Identity)
- Dashboard
- License API
- Dataflow API
- Cloud Instance API
- Discovery API

Every module has its own docker image, and a helm chart for deploying it in a Kubernetes cluster. Once we have the EKS cluster properly configured and the rest of the base components deployed (Kafka, DDBBs, etc), we can deploy 5GMETA modules. For doing that:

````
helm repo add 5gmeta-helm https://<id>@raw.githubusercontent.com/5gmetadmin/helmcharts/main/repository
helm repo update
helm install <name> 5gmeta/<chart_name> -n <organization_namespace>
````

Of course the behaviour of the K8s application can be changed through the values file.

Every time we update the docker image, we have to update the helm deployment, for doing that we can run ``helm upgrade`` commmand, but I suggest uninstalling and installing the chart to be sure all the components are updated.

````
helm uninstall <name> -n <organization_namespace>
helm install <name> 5gmeta/<chart_name> -n <organization_namespace>
````

