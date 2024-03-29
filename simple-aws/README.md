# showcase/simple-aws

Deploying an AWS EKS Cluster and a few sample kubernetes resources to get started.

## Setup and check Astrobase versions

cli
```sh
$ astrobase version
🚀 Astrobase CLI 0.2.1 🧑‍🚀
```

server
```sh
$ astrobase init
$ curl -s -X GET "http://:8787/healthcheck" | jq
{
  "api_version": "v0",
  "api_release_version": "0.1.6",
  "message": "We're on the air.",
  "time": "2021-05-14 20:43:04.924411"
}
```

You may have to run `aws configure` and set your credentials.

## Create EKS Cluster

```sh
$ export CLUSTER_ROLE_ARN=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName == "AstrobaseEKSRole") | .Arn')
$ export NODE_ROLE_ARN=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName == "AstrobaseEKSNodegroupRole") | .Arn')
$ export SUBNET_ID_0=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[0]')
$ export SUBNET_ID_1=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[1]')
$ export SECURITY_GROUP=$(aws ec2 describe-security-groups --query 'SecurityGroups[].GroupId' | jq -r '.[0]')
$ astrobase apply -f cluster.yaml -v "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN NODE_ROLE_ARN=$NODE_ROLE_ARN SUBNET_ID_0=$SUBNET_ID_0 SUBNET_ID_1=$SUBNET_ID_1 SECURITY_GROUP=$SECURITY_GROUP"
{
  "message": "EKS create request submitted for simple"
}
```

Wait a few minutes for the cluster to start up ...

```sh
$ curl -s -X GET "http://:8787/eks/simple?region=us-east-1" | jq '.cluster.status'
"ACTIVE"
```

## Deploy Kubernetes Dashboard and NGINX

```sh
$ astrobase apply -f resources.yaml -v "LOCATION=us-east-1 NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
deployment.apps/nginx-deployment created
```

## Check the NGINX deployment via the dashboard

```sh
$ kubectl proxy
$ open http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/deployment/default/nginx-deployment?namespace=default
```

You may need to retrieve your admin token if using the kubeconfig to authenticate does not work.

```sh
$ aws eks get-token --region us-east-1 --cluster-name simple
```

## Destroy resources

```sh
$ astrobase destroy -f resources.yaml -v "LOCATION=us-east-1 NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
destroying resources in simple@us-east-1
namespace "kubernetes-dashboard" deleted
serviceaccount "kubernetes-dashboard" deleted
service "kubernetes-dashboard" deleted
secret "kubernetes-dashboard-certs" deleted
secret "kubernetes-dashboard-csrf" deleted
secret "kubernetes-dashboard-key-holder" deleted
configmap "kubernetes-dashboard-settings" deleted
role.rbac.authorization.k8s.io "kubernetes-dashboard" deleted
clusterrole.rbac.authorization.k8s.io "kubernetes-dashboard" deleted
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard" deleted
clusterrolebinding.rbac.authorization.k8s.io "kubernetes-dashboard" deleted
deployment.apps "kubernetes-dashboard" deleted
service "dashboard-metrics-scraper" deleted
deployment.apps "dashboard-metrics-scraper" deleted
destroying resources in simple@us-east-1
deployment.apps "nginx-deployment" deleted
```

## Destroy the cluster

```sh
$ astrobase destroy -f cluster.yaml -v "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN NODE_ROLE_ARN=$NODE_ROLE_ARN SUBNET_ID_0=$SUBNET_ID_0 SUBNET_ID_1=$SUBNET_ID_1 SECURITY_GROUP=$SECURITY_GROUP"
{
  "message": "EKS delete request submitted for simple cluster and nodegroups: cpu"
}
```
