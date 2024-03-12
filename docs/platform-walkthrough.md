# 5GMETA Cloud Architecture

## Features:

- 2 worker nodes (1 min / 4 max)
	- T3a.large instances (2vCPUs 8GiB Memory)
	- eu-west-3 region (Paris)
- An Amazon EKS cluster consists of two primary components:
	- The Amazon EKS control plane
	- Amazon EKS nodes that are registered with the control plane
- The Amazon EKS control plane consists of control plane nodes that run the - Kubernetes software, such as etcd and the Kubernetes API server. The control plane runs in an account managed by AWS, and the Kubernetes API is exposed via the Amazon EKS endpoint associated with your cluster.

## Prerequisites

- AWS CLI: [https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- Kubectl: [https://kubernetes.io/docs/tasks/tools/#kubectl](https://kubernetes.io/docs/tasks/tools/)
- Helm: [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)
- AWS EKS Masterclass: [https://github.com/stacksimplify/aws-eks-kubernetes-masterclass](https://github.com/stacksimplify/aws-eks-kubernetes-masterclass), [https://www.stacksimplify.com/aws-eks/](https://www.stacksimplify.com/aws-eks/)

## Platform  access

- Access request: [https://vicomtech.sharepoint.com/:x:/s/5GMETA2/EWkBlYytyP5AvA1A-tbzs3MBhRXfhsrLLddMc6c0mpgrrA?e=TugssA](https://vicomtech.sharepoint.com/:x:/s/5GMETA2/EWkBlYytyP5AvA1A-tbzs3MBhRXfhsrLLddMc6c0mpgrrA?e=TugssA)
- AWS CLI credentials  will be provided  by VICOM: aws configure
- Using AWS CLI and provided  credentials  generate  kubeconfig file with  following  command: ``aws  eks  update-kubeconfig --name 5gmeta-cloud --role-arn  arn:aws:iam::<idaws>:role/k8s5gmeta``
- To change default namespace: ``kubectl config set-context --current --namespace=<my-namespace>``
- The  config file will be created in kubectl default path: ``~/.kube/config``
	- Kubecofig enables kubectl to securely access the Kubernetes Cluster
	- Kubeconfig is a YAML file that contains either a username and password combination or a secure token that when read programmatically removes the need for the Kubernetes client to ask for interactive authentication

## Modules deployment in the cloud platform

Deployment  of apps: Helm charts or K8s manifest files:
- Manifest files:
	- [https://kubernetes.io/docs/tutorials/](https://kubernetes.io/docs/tutorials/)
	- [https://www.mirantis.com/blog/introduction-to-yaml-creating-a-kubernetes-deployment/](https://www.mirantis.com/blog/introduction-to-yaml-creating-a-kubernetes-deployment/)
	- [https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html](https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html)
- Helm Charts:
	- [https://helm.sh/docs/chart_template_guide/getting_started/](https://helm.sh/docs/chart_template_guide/getting_started/)
	- [https://docs.bitnami.com/tutorials/create-your-first-helm-chart/](https://docs.bitnami.com/tutorials/create-your-first-helm-chart/)
	- [https://veducate.co.uk/how-to-create-helm-chart/](https://veducate.co.uk/how-to-create-helm-chart/)
	- [https://jfrog.com/blog/10-helm-tutorials-to-start-your-kubernetes-journey/](https://jfrog.com/blog/10-helm-tutorials-to-start-your-kubernetes-journey/)

All  resources  to be deployed in the  cloud  MUST BE in the  project’s  offical  repositories:
- DockerHub: [https://hub.docker.com/orgs/5gmeta](https://hub.docker.com/orgs/5gmeta)
- Helm Chart repository: [https://github.com/5gmeta/helmcharts](https://github.com/5gmeta/helmcharts)
- GitHub (Manifest files in each module repository):

## Communication  between  pods

- By default, pods can communicate with each other by their IP address, regardless of the namespace they're in. You can see the IP address of each pod with:
- kubectl get pods -o wide –n <my-namespace>
- Decision taken on limits of connectivity across pods
- All pods are "in a LAN“
- No expected access from third-parties to internal IPs ([https://projectcalico.docs.tigera.io/security/calico-network-policy](https://projectcalico.docs.tigera.io/security/calico-network-policy))
- However, the normal way to communicate within a cluster is through Service resources.  A Service also has an IP address and additionally a DNS name. A Service is backed by a set of pods. The Service forwards requests to itself to one of the backing pods. The fully qualified DNS name of a Service is: ``<service-name>.<service-namespace>.svc.cluster.local``
- This can be resolved to the IP address of the Service from anywhere in the cluster (regardless of namespace).
- [https://kubebyexample.com/en/learning-paths/application-development-kubernetes/lesson-3-networking-kubernetes/exposing](https://kubebyexample.com/en/learning-paths/application-development-kubernetes/lesson-3-networking-kubernetes/exposing)

## Persistent storage

- Amazon EBS CSI driver already installed -> Default storage class
- Every Persistent Volume Claim (PVC) will create an AWS EBS Volume
- Volume Integrity is ensured
- More:
	- [https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/](https://loft.sh/blog/kubernetes-persistent-volumes-examples-and-best-practices/)
	- [https://loft.sh/blog/kubernetes-persistent-volumes-examples-and-best-practices/](https://loft.sh/blog/kubernetes-persistent-volumes-examples-and-best-practices/)
	- [https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes)

## Services exposure to the outside world

- NodePort:  Declaring a service as NodePort exposes the Service on each Node’s IP at the NodePort (a fixed port for that Service, in the default range of 30000-32767). You can then access the Service from outside the cluster by requesting <NodeIp>:<NodePort>. Every service you deploy as NodePort will be exposed in its own port, on every Node.
- LoadBalancer:  Declaring a service of type LoadBalancer exposes it externally using a cloud provider’s load balancer. The cloud provider will provision a load balancer for the Service and map it to its automatically assigned NodePort. NLB is created in AWS for each service.
- Ingress:  Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource. ALB is created in AWS. One for all ingress resources.
- More:
	- [https://blog.ovhcloud.com/getting-external-traffic-into-kubernetes-clusterip-nodeport-loadbalancer-and-ingress/](https://blog.ovhcloud.com/getting-external-traffic-into-kubernetes-clusterip-nodeport-loadbalancer-and-ingress/)
	- [https://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/](https://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/)
	- [https://www.eksworkshop.com/beginner/130_exposing-service/ingress/](https://www.eksworkshop.com/beginner/130_exposing-service/ingress/)

## Monitoring  of modules

- Prometheus and Grafana  already  deployed:
	- ``PROMETHEUS=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].metadata.name}')``
	- ``GRAFANA=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}')``
	- ``kubectl port-forward -n monitoring $PROMETHEUS 9090 &``
	- ``kubectl port-forward -n monitoring $GRAFANA 3000 &``

## Pipeline development and deployment

- Flow for deploying a pipeline in an Edge Server:
	1. Receive a request from the cloud platform
	2. Check if enough resources available in the node for the selected SLA
	3. If enough, save the reservation in the DDBB
	4. Instantiate the pipeline through OSM
- Development of a pipeline:
	1. Docker images: [https://hub.docker.com/orgs/5gmeta/repositories](https://hub.docker.com/orgs/5gmeta/repositories)
	2. Helm chart to deploy in K8s: [https://github.com/5gmeta/helmcharts](https://github.com/5gmeta/helmcharts)
	3. OSM descriptors to deploy the Helm chart: [https://github.com/5gmeta/vnfdescriptors](https://github.com/5gmeta/vnfdescriptors)
