# showcase/simple-gcp

Deploying a GCP GKE Cluster and a few sample kubernetes resources to get started.

## Astrobase Versions

cli
```sh
$ astrobase version
🚀 Astrobase CLI 0.1.4 🧑‍🚀
```

server
```sh
$ curl -s -X GET "http://:8787/healthcheck" | jq
{
  "api_version": "v0",
  "api_release_version": "0.1.1",
  "message": "We're on the air.",
  "time": "2021-04-08 04:40:57.630589"
}
```

## Create GKE Cluster

```sh
$ astrobase apply -f cluster.yaml -v "LOCATION=us-central1 PROJECT_ID=$(gcloud config get-value project)"
{
  "name": "operation-1617856937200-eb51a86a",
  "zone": "us-central1",
  "operationType": "CREATE_CLUSTER",
  "status": "RUNNING",
  "selfLink": "https://container.googleapis.com/v1beta1/projects/<hidden>/locations/us-central1/operations/operation-1617856937200-eb51a86a",
  "targetLink": "https://container.googleapis.com/v1beta1/projects/<hidden>/locations/us-central1/clusters/simple",
  "startTime": "2021-04-08T04:42:17.200969776Z"
}
```

Wait a few minutes for the cluster to start up ...

```sh
$ curl -s -X GET "http://:8787/gke?project_id=$(gcloud config get-value project)&location=us-central1" | jq '.clusters[0].status'
"RUNNING"
```

## Deploy Kubernetes Dashboard and NGINX

```sh
$ astrobase apply -f resources.yaml -v "LOCATION=us-central1 NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
applying resources to simple@us-central1
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
applying resources to simple@us-central1
deployment.apps/nginx-deployment created
```

## Check the NGINX deployment via the dashboard

```sh
$ kubectl proxy
$ open http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/deployment/default/nginx-deployment?namespace=default
```

## Destroy resources

```sh
$ astrobase destroy -f resources.yaml -v "LOCATION=us-central1 NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
destroying resources in simple@us-central1
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
destroying resources in simple@us-central1
deployment.apps "nginx-deployment" deleted
```

## Destroy the cluster

```sh
$ astrobase destroy -f cluster.yaml -v "LOCATION=us-central1 PROJECT_ID=astrobase-284118"
{
  "name": "operation-1617858934526-c8422870",
  "zone": "us-central1",
  "operationType": "DELETE_CLUSTER",
  "status": "RUNNING",
  "selfLink": "https://container.googleapis.com/v1beta1/projects/715209933323/locations/us-central1/operations/operation-1617858934526-c8422870",
  "targetLink": "https://container.googleapis.com/v1beta1/projects/715209933323/locations/us-central1/clusters/simple",
  "startTime": "2021-04-08T05:15:34.526759096Z"
}
```
