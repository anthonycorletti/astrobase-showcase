# showcase/databases-aws

- [Set up Astrobase](#set-up-astrobase)
- [Create EKS Cluster](#create-eks-cluster)
- [Deploy Databases](#deploy-databases)
- [Destroy Databases](#destroy-databases)
- [Destroy EKS cluster](#destroy-eks-cluster)

Deploy Apache Druid, Postgres, and ClickHouse to EKS.

## Set up Astrobase

### Install the CLI

```sh
$ python -m pip install astrobase_cli
$ astrobase version
🚀 Astrobase CLI 0.1.9 🧑‍🚀
```

### Check Command Prerequisites

For this example, you will only need `aws`, `docker`, and `kubectl` available in your shell.

```sh
$ astrobase check commands
az found at: /usr/local/bin/az
aws found at: /usr/local/bin/aws
docker found at: /usr/local/bin/docker
gcloud found at: /usr/local/google-cloud-sdk/bin/gcloud
kubectl found at: /usr/local/bin/kubectl
```

### Create your astrobase profile

Our profile will only configure AWS here. If you would like to configure more clouds, [follow the instructions in the features and usage section of the Astrobase CLI documentation](https://github.com/astrobase/cli/blob/master/README.md#features-and-usage).

```sh
$ astrobase profile create database-aws \
--aws-creds /usr/local/.aws/credentials \
--aws-profile-name default
Created profile database-aws.
$ export ASTROBASE_PROFILE=database-aws
$ astrobase profile current
{
  "name": "database-aws",
  "server": "http://localhost:8787",
  "gcp_creds": null,
  "aws_creds": "/usr/local/.aws/credentials",
  "aws_profile_name": "default",
  "azure_client_id": null,
  "azure_client_secret": null,
  "azure_subscription_id": null,
  "azure_tenant_id": null
}
```

### Initialize Astrobase

```sh
$ astrobase init
Initializing Astrobase ...
Starting Astrobase server ...
Astrobase initialized and running at http://localhost:8787
```

```sh
$ astrobase version && curl -s "http://:8787/healthcheck" | jq .
🚀 Astrobase CLI 0.1.9 🧑‍🚀
{
  "api_version": "v0",
  "api_release_version": "0.1.5",
  "message": "We're on the air.",
  "time": "2021-05-02 01:59:11.552895"
}
```

## Create EKS Cluster

### Create Necessary Roles for EKS

You will need to create 2 roles to operate EKS, a cluster role, and a node role.

[Follow these instructions for creating the __cluster__ role.](https://github.com/astrobase/cli#cluster_role_arn)

[Follow these instructions for creating the __node__ role.](https://github.com/astrobase/cli#node_role_arn)

### Set Necessary Values

```sh
export CLUSTER_ROLE_ARN_NAME="AstrobaseEKSRole"
export NODE_ROLE_ARN_NAME="AstrobaseEKSNodegroupRole"
export CLUSTER_ROLE_ARN=$(aws iam list-roles | jq --arg CLUSTER_ROLE_ARN_NAME "$CLUSTER_ROLE_ARN_NAME" -r '.Roles[] | select(.RoleName == $CLUSTER_ROLE_ARN_NAME) | .Arn')
export NODE_ROLE_ARN=$(aws iam list-roles | jq --arg NODE_ROLE_ARN_NAME "$NODE_ROLE_ARN_NAME" -r '.Roles[] | select(.RoleName == $NODE_ROLE_ARN_NAME) | .Arn')
export SUBNET_ID_0=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[0]')
export SUBNET_ID_1=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId[]' | jq -r '.[1]')
export SECURITY_GROUP=$(aws ec2 describe-security-groups --query 'SecurityGroups[].GroupId' | jq -r '.[0]')
```

### Send a request to create the cluster and monitor cluster and nodegroup statuses

```sh
$ git clone https://github.com/astrobase/showcase && cd showcase
$ cd databases-aws
$ astrobase apply -f cluster.yaml -v "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN NODE_ROLE_ARN=$NODE_ROLE_ARN SUBNET_ID_0=$SUBNET_ID_0 SUBNET_ID_1=$SUBNET_ID_1 SECURITY_GROUP=$SECURITY_GROUP"
{
  "message": "EKS create request submitted for database-testing"
}
```

```sh
$ curl -s -X GET "http://:8787/eks/database-testing?region=us-east-1" | jq '.cluster.status'
"CREATING"
```

EKS is notorious for taking a while to create nodegroups, so it may take 15-20 minutes to create the cluster. Good that you're using astrobase huh? Imagine doing lots of complex ops work just to wait 20 minutes. Not with Astrobase!

```sh
$ curl -s -X GET "http://:8787/eks/database-testing?region=us-east-1" | jq '.cluster.status'
"ACTIVE"
```

Now check the nodegroup

```sh
$ curl -s -X GET "http://:8787/eks/database-testing/nodegroups/database-testing?region=us-east-1" | jq '.nodegroup.status'
"CREATING"
```

```sh
$ curl -s -X GET "http://:8787/eks/database-testing/nodegroups/database-testing?region=us-east-1" | jq '.nodegroup.status'
"ACTIVE"
```

## Deploy Databases

```sh
$ astrobase apply -f databases.yaml
```

If the command fails, simply re-run it. It takes an extra second or two for kubernetes to provision custom resource definitions for Druid and Clickhouse.

### Connect to Postgres

```sh
$ kubectl port-forward svc/postgres 5432:5432 &
$ psql -h localhost -U postgres --password -p 5432
Password:
psql (13.2, server 10.4 (Debian 10.4-2.pgdg90+1))
Type "help" for help.

postgres=# \dt
Did not find any relations.
```

### Connect to Druid

```sh
$ kubectl port-forward svc/druid-tiny-cluster-routers 8088:8088 &
$ open http://localhost:8088/
```

### Connect to Clickhouse

```sh
$ export CLICKHOUSE_LOADBALANCER=$(kubectl get svc --field-selector metadata.name=clickhouse-pv-simple -o json | jq -r '.items[0].status.loadBalancer.ingress[0].hostname')
$ docker run -it yandex/clickhouse-client -h $CLICKHOUSE_LOADBALANCER -u clickhouse_operator --password clickhouse_operator_password
ClickHouse client version 21.4.6.55 (official build).
Connecting to a14502ebbff0d4f4d97f0d9c42ecac33-354106546.us-east-1.elb.amazonaws.com:9000 as user clickhouse_operator.
Connected to ClickHouse server version 21.4.6 revision 54447.

chi-pv-simple-shards-0-0-0.chi-pv-simple-shards-0-0.default.svc.cluster.local :)
```

## Destroy EKS cluster

```sh
$ astrobase destroy -f cluster.yaml -v "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN NODE_ROLE_ARN=$NODE_ROLE_ARN SUBNET_ID_0=$SUBNET_ID_0 SUBNET_ID_1=$SUBNET_ID_1 SECURITY_GROUP=$SECURITY_GROUP"
```

