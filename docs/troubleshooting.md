# Platform reboot
If k8s nodes are rebooted, some ips must be modified

## Get new public ips from k8s nodes

Go to: [https://eu-west-3.console.aws.amazon.com/eks/home?region=eu-west-3#/clusters/5gmeta-cloud?selectedTab=cluster-compute-tab](https://eu-west-3.console.aws.amazon.com/eks/home?region=eu-west-3#/clusters/5gmeta-cloud?selectedTab=cluster-compute-tab)

Go to console Amazon web page -> EKS -> click on 5gmeta-cloud -> compute tab
Open each compute and click on instance. There you can find public IP

## Add ips to security groups in Amazon EKS
Add new ips in pl-0f53017bf8a4dc952 prefix list: [https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#PrefixListDetails:PrefixListId=pl-0f53017bf8a4dc952](https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#PrefixListDetails:PrefixListId=pl-0f53017bf8a4dc952)
Check that pl-0f53017bf8a4dc952 prefix list is allowed to access 61616 TCP port in security group sg-07e47c7047324400b [https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#SecurityGroup:groupId=sg-07e47c7047324400b](https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#SecurityGroup:groupId=sg-07e47c7047324400b)

## Reconfigure Kafka cluster
Add new ips in [https://github.com/5gmeta/stream-data-gateway/blob/main/src/prod-version/cp-helm-charts/charts/cp-kafka/values.yaml]

And reinstall helm chart:

* helm get values kafkacluster -n kafka
* helm uninstall kafkacluster -n kafka
* kubectl delete namespace kafka
* from stream-data-gateway/src/prod-version git folder
	* helm install kafkacluster ./cp-helm-charts --create-namespace -n kafka

## Reconfigure Registration API

Update access to 5GMETA database: Add one of k8s nodes new ip address in [https://github.com/5gmeta/helmcharts/blob/main/charts/registrationapi-chart/values.yaml](https://github.com/5gmeta/helmcharts/blob/main/charts/registrationapi-chart/values.yaml)

Rebuild registration API:
- helm package ./charts/* -d ./repository/
- helm repo index ./repository/
- Push the changes to the repository

Ask parters to update Helm charts from their MECS:

```
        Log in into your MEC
        Update 5GMETA helm repositories
            helm repo update 5gmeta
        Uninstall previous registration-api helm
            helm uninstall registration-api -n 5gmeta
        Install updated registration-api helm
            helm install registration-api 5gmeta/registrationapi-chart  -n 5gmeta
```

## Update all examples and 


## Add load balancer ips to allowed ips

Ping to 5gmeta-platform.eu or to DNS hostname (get from https://eu-west-3.console.aws.amazon.com/ec2/home?region=eu-west-3#LoadBalancers:). It will return 3 ips. Copy them into pl-0f53017bf8a4dc952 prefix list: [https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#PrefixListDetails:PrefixListId=pl-0f53017bf8a4dc952](https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#PrefixListDetails:PrefixListId=pl-0f53017bf8a4dc952)

## Reboot Load balancer (APISIX/ GATEWAY)

If new whitelisted ip addresses are needed, a new prefix list ([https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#ManagedPrefixLists:](https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#ManagedPrefixLists:)) with them should be filled. That list must have access to the ports 80 (HTTP) and 443 (HTTPS). To do that we have to create a new security group that points to new prefix list.

Best option is to copy [https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#SecurityGroup:groupId=sg-06755943a0511d718](https://eu-west-3.console.aws.amazon.com/vpc/home?region=eu-west-3#SecurityGroup:groupId=sg-06755943a0511d718) to a new rule and add prefix list as source ip addresses. 

Add that created security to [https://github.com/5gmeta/helmcharts/blob/main/charts/gateway-chart/values.yaml] and rebuild helm chart and uninstall/install from EKS.


```
alb.ingress.kubernetes.io/security-groups: sg-06755943a0511d718,new_security_group
```

### Rebuild Helm charts

* helm package ./charts/* -d ./repository/
* helm repo index ./repository/
* Push the changes to the repository

### Uninstall / Install Gateway

* helm uninstall apisix-service -n akka
* helm install apisix-service 5gmeta-helm/gateway-standalone -n akka


### Update 5gmeta-platform.eu hostname

When a new Load balancer is deployed, DNS hostname is modified, so it must be changed in Amazon DNS manager:

To check Load balancer hostname go to [[https://eu-west-3.console.aws.amazon.com/ec2/home?region=eu-west-3#LoadBalancers:](https://eu-west-3.console.aws.amazon.com/ec2/home?region=eu-west-3#LoadBalancers:) and replace it in [https://us-east-1.console.aws.amazon.com/route53/v2/home#Dashboard](https://us-east-1.console.aws.amazon.com/route53/v2/home#Dashboard).
Edit hosted zones and modify A record from 5gmeta-platform.eu adding DNS hosted zone +dualstack: dualstack.k8s-5gmetaingress-253cf23a0c-1635503379.eu-west-3.elb.amazonaws.com for example.




 
