# Multi-cloud Deployments

Deploying services across many clouds is easy with Astrobase.

Follow the [simple-gcp](./simple-gcp#create-gke-cluster) and [simple-aws](./simple-aws#create-eks-cluster) instructions for creating clusters only, then run the following command to deploy services across both clouds

```sh
$ astrobase apply -f resources.yaml -v "EKS_LOCATION=us-east-1 GKE_LOCATION=us-central1-c NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
applying resources to simple@us-east-1
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
applying resources to simple@us-east-1
deployment.apps/nginx-deployment created
applying resources to simple@us-central1-c
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
applying resources to simple@us-central1-c
deployment.apps/nginx-deployment created
```
