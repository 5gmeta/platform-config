# Cloud platform deployment

For deploying 5GMETA Cloud Platform, AWS EKS solution has been chosen. Amazon Elastic Kubernetes Service ([Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)) is a managed service that you can use to run Kubernetes on AWS without needing to install, operate, and maintain your own Kubernetes control plane or nodes. Kubernetes is an open-source system for automating the deployment, scaling, and management of containerized applications.

Amazon EKS runs a single tenant Kubernetes control plane for each cluster. The control plane infrastructure isn't shared across clusters or AWS accounts. The control plane consists of at least two API server instances and three `etcd` instances that run across three Availability Zones within an AWS Region. The control plane runs in an account managed by AWS, and the Kubernetes API is exposed via the Amazon EKS endpoint associated with your cluster. Each Amazon EKS cluster control plane is single-tenant and unique, and runs on its own set of Amazon EC2 instances.

The workloads are deployed in Amazon EKS nodes that are registered with the control plane. They are deployed in EC2 service.

The following picture shows the architecurte of an AWS EKS cluster:

![EKS Architecture](img/EKS%20Architecture.png)

## Prerequisites

Many procedures of the guide use the following command line tools:

-   `kubectl`  – A command line tool for working with Kubernetes clusters. For more information, see  [Installing or updating kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).
	````
	curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl

	curl -LO https://dl.k8s.io/release/v1.21.0/bin/linux/amd64/kubectl

	sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

	kubectl version --client
	````
-   `eksctl`  – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see  [Installing or updating eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html).
	````
	curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

	sudo mv /tmp/eksctl /usr/local/bin

	eksctl version
	````
    
-   `AWS CLI`  – A command line tool for working with AWS services, including Amazon EKS. For more information, see  [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)  in the AWS Command Line Interface User Guide. After installing the AWS CLI, we recommend that you also configure it. For more information, see  [Quick configuration with  `aws configure`](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)  in the AWS Command Line Interface User Guide.
	````
	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

	unzip awscliv2.zip

	sudo ./aws/install

	aws –version

	aws configure
	````
-   `helm`  – Helm helps you manage Kubernetes applications — Helm Charts help you define, install, and upgrade even the most complex Kubernetes application. For more information, see [Installing Helm](https://helm.sh/docs/intro/install/).
	````
	curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

	chmod 700 get_helm.sh

	./get_helm.sh

	helm version
	````

## Step 1: Create Amazon EKS cluster and nodes

Before creating a cluster and nodes for production use, familiarize yourself with all settings and deploy a cluster and nodes with the settings that meet your requirements. For more information, see  [Creating an Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)  and  [Amazon EKS nodes](https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html). Some settings can only be enabled when creating your cluster and nodes.

Create your Amazon EKS cluster with the following command.
````
eksctl create cluster -f 5gmeta-cluster-config.yaml

eksctl get cluster

aws eks describe-cluster --name 5gmeta-cloud

helm repo add 5gmeta https://<id>@raw.githubusercontent.com/5gmetadmin/helmcharts/main/repository

kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<user> --docker-password=<password>
````

The yaml configuration file can be found in this repository: [5gmeta-cluster-config.yaml](https://github.com/5gmeta/platform-config/blob/main/src/eksctl/5gmeta-cluster-config.yaml)

The cluster that will be created will be made up of two nodes, and can be scaled up to 4 nodes without the cluster running out of resources. The nodes will be deployed in the eu-west-3 region, with Kubernetes version 1.21 and with the instance type t3a.large (2 vCPU, 8GiB Memory). [5gmeta-cluster-ssh-key.pem](https://github.com/5gmeta/platform-config/blob/main/src/5gmeta-cluster-ssh-key.pem) key is used for ssh access. It was previously created using EC2 AWS [interface](https://eu-west-3.console.aws.amazon.com/ec2/home?region=eu-west-3#KeyPairs:).

To avoid unneeded costs when nodes are going to be in public nodegroups anyway,  default NAT gateway creation is disabled. Automatic creation of IAM Roles for autoScaler, ebs and albIngress are enabled.

````
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: 5gmeta-cloud
  region: eu-west-3
  version: "1.21"

vpc:
  nat:
    gateway: Disable # other options: HighlyAvailable, Single

iam:
  withOIDC: true

managedNodeGroups:
  - name: managed-ng-1
    instanceType: t3a.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    volumeSize: 50
    ssh:
      allow: true # will use ~/.ssh/id_rsa.pub as the default ssh key if no publicKeyNamem specified
      publicKeyName: 5gmeta-cluster-key # key created in aws
    iam:
      withAddonPolicies:
        autoScaler: true
        ebs: true
        albIngress: true
    ami: ami-<id> # get ami id 'aws ssm get-parameter --name /aws/service/eks/optimized-ami/1.21/amazon-linux-2/recommended/image_id --region eu-west-3 --query "Parameter.Value" --output text'
    overrideBootstrapCommand: |
      #!/bin/bash
      /etc/eks/bootstrap.sh 5gmeta-cloud --kubelet-extra-args "--kube-reserved memory=0.3Gi,ephemeral-storage=1Gi --system-reserved memory=0.3Gi,ephemeral-storage=1Gi --eviction-hard memory.available<200Mi,nodefs.available<10%"
````

## Step 2: Configure Amazon EKS cluster

**Create an IAM OIDC provider:**
Amazon EKS supports using OpenID Connect (OIDC) identity providers as a method to authenticate users in the cluster. After configuring authentication in the cluster, Kubernetes `roles` and `clusterroles` can be created to assign permissions to the roles, and then bind the roles to the identities using Kubernetes `rolebindings` and `clusterrolebindings`.

The cluster has an [OpenID Connect](https://openid.net/connect/) (OIDC) issuer URL associated with it. To use AWS Identity and Access Management (IAM) roles for service accounts, an IAM OIDC provider must exist.
````
oidc_id=$(aws eks describe-cluster --name 5gmeta-cloud --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

aws iam list-open-id-connect-providers | grep $oidc_id
````
If output is returned from the previous command, a provider for the cluster is already created and next step can be skipped. If no output is returned, then an IAM OIDC provider must be created.
````
eksctl utils associate-iam-oidc-provider --cluster 5gmeta-cloud --approve
````

**Install AWS Load Balancer Controller add-on:**

The AWS Load Balancer Controller manages AWS Elastic Load Balancers for a Kubernetes cluster. The controller provisions the following resources.
- An AWS Application Load Balancer (ALB) when you create a Kubernetes Ingress.
- An AWS Network Load Balancer (NLB) when you create a Kubernetes service of type LoadBalancer. In the past, the Kubernetes network load balancer was used for instance targets, but the AWS Load balancer Controller was used for IP targets. With the AWS Load Balancer Controller version 2.3.0 or later, you can create NLBs using either target type. For more information about NLB target types, see Target type in the User Guide for Network Load Balancers.

The AWS Load Balancer Controller was formerly named the _AWS ALB Ingress Controller_. It's an [open-source project](https://github.com/kubernetes-sigs/aws-load-balancer-controller) managed on GitHub. This topic describes how to install the controller using default options. You can view the full [documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/) for the controller on GitHub. Before deploying the controller, we recommend that you review the prerequisites and considerations in [Application load balancing on Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html) and [Network load balancing on Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html). Those topics also include steps on how to deploy a sample application that require the AWS Load Balancer Controller to provision AWS Application Load Balancers and Network Load Balancers.

To deploy the AWS Load Balancer Controller to an Amazon EKS cluster:

````
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.1/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

eksctl create iamserviceaccount --cluster=5gmeta-cloud --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::<idaws>:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve #<idaws> refers to the account ID

helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=5gmeta-cloud --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller
````

**Install AWS Storage Controller**

Amazon EKS clusters that were created prior to Kubernetes version `1.11` weren't created with any storage classes. You must define storage classes for your cluster to use and you should define a default storage class for your persistent volume claims. For more information, see [Storage classes](https://kubernetes.io/docs/concepts/storage/storage-classes) in the Kubernetes documentation.

The Amazon Elastic Block Store (Amazon EBS) Container Storage Interface (CSI) driver allows Amazon Elastic Kubernetes Service (Amazon EKS) clusters to manage the lifecycle of Amazon EBS volumes for persistent volumes. The Amazon EBS CSI driver isn't installed when you first create a cluster.

More information about `ebs.csi.aws.com` and `kubernetes.io/aws-ebs` provisioners:
[https://stackoverflow.com/questions/68359043/whats-the-difference-between-ebs-csi-aws-com-vs-kubernetes-io-aws-ebs-for-provi](https://stackoverflow.com/questions/68359043/whats-the-difference-between-ebs-csi-aws-com-vs-kubernetes-io-aws-ebs-for-provi)

````
curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json

aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://example-iam-policy.json

eksctl create iamserviceaccount --name ebs-csi-controller-sa --namespace kube-system --cluster 5gmeta-cloud --attach-policy-arn arn:aws:iam::<idaws>:policy/AmazonEKS_EBS_CSI_Driver_Policy --approve --override-existing-serviceaccounts #<idaws> refers to the account ID

helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update

helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver --namespace kube-system --set controller.serviceAccount.create=false --set controller.serviceAccount.name=ebs-csi-controller-sa

kubectl get deployment -n kube-system ebs-csi-controller

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/examples/kubernetes/dynamic-provisioning/manifests/storageclass.yaml

kubectl get storageclass

kubectl patch storageclass gp2 -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

kubectl patch storageclass ebs-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
````

**Define the default storage class**
A StorageClass provides a way for administrators to describe the "classes" of storage they offer. [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) allows storage [volumes](https://kubernetes.io/docs/concepts/storage/volumes/)  and [persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes) to be created on-demand. [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) should be pre-created to define which provisioner should be used and what parameters should be passed when dynamic provisioning is invoked.

To define the storage class the following [manifest file](https://github.com/5gmeta/platform-config/blob/main/src/storageclass/storageclass-retain.yaml) will be applied. Thanks to this storage class every persistent volume will be deployed in AWS S3, using the previously installed storage controller.

````
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-retain
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
````

To become the storage class the default one: ``kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class=true``

**Install Cluster Autoscaler**

The Kubernetes [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) automatically adjusts the number of nodes in your cluster when pods fail or are rescheduled onto other nodes. The Cluster Autoscaler is typically installed as a [Deployment](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws/examples) in your cluster. It uses [leader election](https://en.wikipedia.org/wiki/Leader_election) to ensure high availability, but scaling is done by only one replica at a time.

Create an IAM policy that grants the permissions that the Cluster Autoscaler requires to use an IAM role. Save the following contents to a file that's named `cluster-autoscaler-policy.json`.
````
{
	"Version": "2012-10-17",
	"Statement": [
		{
		"Effect": "Allow",
		"Action": [
			"autoscaling:DescribeAutoScalingGroups",
			"autoscaling:DescribeAutoScalingInstances",
			"autoscaling:DescribeLaunchConfigurations",
			"autoscaling:DescribeTags",
			"ec2:DescribeInstanceTypes",
			"ec2:DescribeLaunchTemplateVersions"
		],
		"Resource": ["*"]
		},
		{
		"Effect": "Allow",
		"Action": [
			"autoscaling:SetDesiredCapacity",
			"autoscaling:TerminateInstanceInAutoScalingGroup"
		],
		"Resource": ["*"]
		}
	]
}
````

Create the policy with the following command.
````
aws iam create-policy --policy-name AmazonEKSClusterAutoscalerPolicy --policy-document file://cluster-autoscaler-policy.json

eksctl create iamserviceaccount --cluster=5gmeta-cloud --namespace=kube-system --name=cluster-autoscaler --attach-policy-arn=arn:aws:iam::<idaws>:policy/AmazonEKSClusterAutoscalerPolicy --override-existing-serviceaccounts --approve #<idaws> refers to the account ID
````

Complete the following steps to deploy the Cluster Autoscaler. We recommend that you review [Deployment considerations](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#ca-deployment-considerations) and optimize the Cluster Autoscaler deployment before you deploy it to a production cluster.
````
curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
````

Edit the cluster-autoscaler manifest to replace <YOUR CLUSTER NAME> (including <>) with the name of your cluster, and add the following options. --balance-similar-node-groups ensures that there is enough available compute across all availability zones. --skip-nodes-with-system-pods=false ensures that there are no problems with scaling to zero.

````
kubectl apply -f cluster-autoscaler-autodiscover.yaml

kubectl annotate serviceaccount cluster-autoscaler -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::<idaws>:role/AmazonEKSClusterAutoscalerRole --aprove #<idaws> refers to the account ID

kubectl patch deployment cluster-autoscaler -n kube-system -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'

export K8S_VERSION=$(kubectl version --short | grep 'Server Version:' | sed 's/[^0-9.]*\([0-9.]*\).*/\1/' | cut -d. -f1,2)

export AUTOSCALER_VERSION=$(curl -s "https://api.github.com/repos/kubernetes/autoscaler/releases" | grep '"tag_name":' | sed -s 's/.*-\([0-9][0-9\.]*\).*/\1/' | grep -m1 ${K8S_VERSION})

kubectl set image deployment cluster-autoscaler -n kube-system cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v${AUTOSCALER_VERSION}

kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
````

**Modify AWS Security Group for accessing NodePorts (30000-32768) from outside (if needed)**
