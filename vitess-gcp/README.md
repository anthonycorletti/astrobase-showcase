# showcase/simple-gcp

Deploying a Vitess Clusters on GCP GKE.

## Astrobase Versions

```sh
$ python -m pip install astrobase_cli
$ astrobase version
🚀 Astrobase CLI 0.2.1 🧑‍🚀
$ astrobase profile create your-profile-name --gcp-creds /path/to/gcp_creds.json
$ export ASTROBASE_PROFILE="your-profile-name"
$ curl -s "http://:8787/healthcheck" | jq
{
  "api_version": "v0",
  "api_release_version": "0.1.7",
  "message": "We're on the air.",
  "time": "2021-05-29 19:28:32.857380"
}
```

## Other requirements

Install [`vtctlclient`](https://vitess.io/docs/get-started/operator/).

```sh
$ go get vitess.io/vitess/go/cmd/vtctlclient
```

Install [`mysql`](https://dev.mysql.com/doc/mysql-installation-excerpt/8.0/en/).

## Create GKE Cluster

```sh
$ astrobase apply -f cluster.yaml -v "LOCATION=us-central1-a PROJECT_ID=$(gcloud config get-value project)"
{
  "name": "operation-1622316958998-f254e191",
  "zone": "us-central1-a",
  "operationType": "CREATE_CLUSTER",
  ...
}
```

```sh
$ curl -s -X GET "http://:8787/gke/astrobase-vitess?project_id=$(gcloud config get-value project)&location=us-central1-a" | jq .status
"RUNNING"
```

## Setup and Deploy Vitess (with backups!)

### Create GCS Bucket

You have to use a globally unique name

```sh
$ export VITESS_BACKUPS_NAME="vitess-backups-$(date | sha256sum | head -c 7)"
$ gsutil mb gs://${VITESS_BACKUPS_NAME}
$ gcloud container clusters get-credentials astrobase-vitess --zone us-central1-a
$ kubectl create secret generic $VITESS_BACKUPS_NAME --from-file=$(astrobase profile current | jq -r .gcp_creds)
```

```sh
$ astrobase apply -f vitess.yaml -v "PROJECT_ID=$(gcloud config get-value project) LOCATION=us-central1-a BACKUP_GCS_BUCKET_NAME=$VITESS_BACKUPS_NAME BACKUP_GCS_SECRET_NAME=$VITESS_BACKUPS_NAME BACKUP_GCS_SECRET_KEY=$(astrobase profile current | jq -r .gcp_creds | xargs basename) MYSQL_USER=my-user MYSQL_PASSWORD=my-password"
```

If the first apply returns an error like `no matches for kind "VitessCluster" in version "planetscale.com/v2"`, you should re-run the command. Sometimes CustomResourceDefinitions take a few extra seconds to provision successfully. This is something we're actively looking into.

Run a port forward to Vitess and apply a vschema and mysql schema to the Vitess database.

```
$ kubectl port-forward --address localhost deployment/$(kubectl get deployment --selector="planetscale.com/component=vtctld" -o=jsonpath="{.items..metadata.name}") 15999:15999
$ vtctlclient -server localhost:15999 ApplyVSchema -vschema "$(cat ./schemas/vschema.json)" main
$ vtctlclient -server localhost:15999 ApplySchema -sql "$(cat ./schemas/schema.sql)" main
```

Expose vitess over a service.

```sh
$ kubectl expose deployment $(kubectl get deployment --selector="planetscale.com/component=vtgate" -o=jsonpath="{.items..metadata.name}") --type=LoadBalancer --name=test-vtgate --port 3306 --target-port 3306
$ kubectl get service test-vtgate
```

Get the external ip from the get service/test-vtgate command.

## Test Vitess

```sh
$ mysql -u my-user -h [external-ip] -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.7.9-vitess-10.0.1 Version: 10.0.1 (Git revision f7304cd18 branch 'heads/v10.0.1') built on Tue May  4 12:34:04 UTC 2021 by vitess@dd528fffc22e using go1.15.6 linux/amd64

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SHOW VSCHEMA TABLES;
+----------------+
| Tables         |
+----------------+
| dual           |
| users          |
| users_name_idx |
+----------------+
3 rows in set (0.03 sec)
```

🙌

## Destroy Cluster

```sh
$ astrobase destroy -f cluster.yaml -v "LOCATION=us-central1-a PROJECT_ID=$(gcloud config get-value project)"
```
