# Kubernetes Cluster Access Grant

## Adding a new user

To add a new user in an already configured EKS cluster:

1. Create the user in [AWS EKS](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/users)
	- Username: user's e-mail address
	- Add it to **k8s5gmeta** group
	- Create **Access keys** for the user
		- Go to the user
		- Security credentials tab
		- Create access key
		- Check Command Line Interface
		- Download csv file
	- Save the generated credentiales in [AWS Users](https://github.com/5gmeta/platform-config/tree/main/src/credentials/AWS%20Users)
2. Create a namespace for the user's company (If it is not already created): ``kubectl create namespace <COMPANY>``
3. Apply the [5gmeta-cluster-access.yaml]() manifest file to the namespace: ``kubectl apply -f 5gmeta-cluster-access.yaml -n <COMPANY>``. To get all the namespaces that have this role (the namespaces the new user will have access to) run: ``kubectl get role -A | grep 5gmeta-access-role ``
4. Send the credentials to the user and inform how to generate the kubeconfig file. Sample mail: 
	````
	Dear XXX,

	I am sending you the credentials for accessing the cloud platform:

	https://onetimesecret.com/

	You can only open the link once, so save the credentials locally :)

	You need to install aws cli version2.7.29:
	* curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.7.29.zip" -o "awscliv2.zip"*
	* unzip awscliv2.zip
	* sudo ./aws/install (if you have a previous version installed -> sudo ./aws/install --update
	
	and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) version 1.26.0.
 	* curl -LO https://dl.k8s.io/release/v1.26.0/bin/linux/amd64/kubectl
 	* sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
 
  	Once installed sign-in into AWS with the following command: aws configure. IMPORTANT, select “eu-west-3” region and "text" as output format.

	Then generate the kubeconfig file with the following command: aws eks update-kubeconfig --name 5gmeta-cloud --role-arn arn:aws:iam::<idaws>:role/k8s5gmeta

	After running this command, the kubeconfig file is generated and you will be able to access the cloud cluster with kubectl.

	You have access to <COMPANY> namespace. Please configure it as the default namespace in kubectl with the following command: kubectl config set-context --current --namespace=<COMPANY>

	Please do not hesitate to contact me if you have any doubt.

	Regards,

	XXX
	````

The steps to generate the kubeconfug file:
````
aws configure
aws eks update-kubeconfig --name 5gmeta-cloud --role-arn arn:aws:iam::<idaws>:role/k8s5gmeta
kubectl config set-context --current --namespace=<COMPANY>
````

## Configuration of the cluster

For deploying all the necessary stuff to add users to an EKS hosted K8s cluster, the following tutorial has been followed: [https://www.eksworkshop.com/beginner/091_iam-groups/](https://www.eksworkshop.com/beginner/091_iam-groups/)

Run the following commands:
````
aws iam create-group --group-name k8sAdmin
aws iam create-group --group-name k8s5gmeta
ACCOUNT_ID=<idaws>
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')
echo ACCOUNT_ID=$ACCOUNT_ID
echo POLICY=$POLICY
aws iam create-role --role-name k8sAdmin --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." --assume-role-policy-document "$POLICY" --output text --query 'Role.Arn'
aws iam create-role --role-name k8s5gmeta --description "5gmeta role for partner access (for AWS IAM Authenticator for Kubernetes)." --assume-role-policy-document "$POLICY" --output text --query 'Role.Arn'

ADMIN_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8sAdmin"
    },
    {
      "Sid": "AllowEksDescribeCluster",
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "arn:aws:eks:eu-west-3:<idaws>:cluster/5gmeta-cloud"
    }
  ]
}')
echo ADMIN_GROUP_POLICY=$ADMIN_GROUP_POLICY

aws iam put-group-policy \
--group-name k8sAdmin \
--policy-name k8sAdmin-policy \
--policy-document "$ADMIN_GROUP_POLICY"

5GMETA_GROUP_POLICY=$(echo -n '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeOrganizationAccountRole",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':role/k8s5gmeta"
    },
    {
      "Sid": "AllowEksDescribeCluster",
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "arn:aws:eks:eu-west-3:<idaws>:cluster/5gmeta-cloud"
    }
  ]
}')
echo 5GMETA_GROUP_POLICY=$5GMETA_GROUP_POLICY

aws iam put-group-policy --group-name k8s5gmeta --policy-name k8s5gmeta-policy --policy-document "$5GMETA_GROUP_POLICY"
aws iam list-groups
eksctl create iamidentitymapping --cluster 5gmeta-cloud --arn arn:aws:iam::${ACCOUNT_ID}:role/k8sAdmin --username admin --group system:masters
eksctl create iamidentitymapping --cluster 5gmeta-cloud --arn arn:aws:iam::${ACCOUNT_ID}:role/k8s5gmeta --username <username>
kubectl get cm -n kube-system aws-auth -o yaml
eksctl get iamidentitymapping --cluster 5gmeta-cloud
````

Create [5gmeta-cluster-access.yaml](https://github.com/5gmeta/platform-config/blob/main/src/cluster-admin/5gmeta-cluster-access.yaml) manifest file and apply it to namespaces that will access k8s5gmeta group users:

````
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: 5gmeta-access-role
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
      - "autoscaling"
    resources: ["*"]
#    resources:
#      - "configmaps"
#      - "cronjobs"
#      - "deployments"
#      - "events"
#      - "ingresses"
#      - "jobs"
#      - "pods"
#      - "pods/attach"
#      - "pods/exec"
#      - "pods/log"
#      - "pods/portforward"
#      - "secrets"
#      - "services"
    verbs: ["*"]
#    verbs:
#      - "create"
#      - "delete"
#      - "describe"
#      - "get"
#      - "list"
#      - "patch"
#      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: 5gmeta-role-binding
subjects:
- kind: User
  name: <username>
roleRef:
  kind: Role
  name: 5gmeta-access-role
  apiGroup: rbac.authorization.k8s.io
````

To get all the namespaces that have this role (the namespaces the new user will have access to) run: ``kubectl get role -A | grep 5gmeta-access-role ``

