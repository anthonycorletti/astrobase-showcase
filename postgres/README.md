# showcase/postgres

Deploy a distributed PostgresQL cluster on EKS.

## Setup and check Astrobase versions

```sh
astrobase version
🚀 Astrobase CLI 0.2.4 🧑‍🚀
```

```sh
astrobase init
curl -s -X GET "http://:8787/healthcheck" | jq
{
  "api_version": "v0",
  "api_release_version": "0.1.7",
  "message": "We're on the air.",
  "time": "2021-11-21 20:22:15.431026"
}
```

You may have to run `aws configure` and set your credentials.

## Create EKS Cluster

```sh
export CLUSTER_ROLE_ARN=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName == "AstrobaseEKSRole") | .Arn')
export NODE_ROLE_ARN=$(aws iam list-roles | jq -r '.Roles[] | select(.RoleName == "AstrobaseEKSNodegroupRole") | .Arn')
export SUBNET_ID_0=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[0]')
export SUBNET_ID_1=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[1]')
export SECURITY_GROUP=$(aws ec2 describe-security-groups --query 'SecurityGroups[].GroupId' | jq -r '.[0]')
astrobase apply -f cluster.yaml -v "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN NODE_ROLE_ARN=$NODE_ROLE_ARN SUBNET_ID_0=$SUBNET_ID_0 SUBNET_ID_1=$SUBNET_ID_1 SECURITY_GROUP=$SECURITY_GROUP"
{
  "message": "EKS create request submitted for postgres-eks"
}
```

Wait a few minutes for the cluster to start up ...

```sh
curl -s -X GET "http://:8787/eks/postgres-eks?region=us-east-1" | jq '.cluster.status'
"ACTIVE"
```

## Deploy Postgres

```sh
astrobase apply -f resources.yaml
```


## Destroy resources

```sh
astrobase destroy -f resources.yaml
```

## Destroy the cluster

```sh
astrobase destroy -f cluster.yaml
```
