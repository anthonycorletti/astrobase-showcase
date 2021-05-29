# Multi-cloud Deployments

Deploying services across many clouds is easy with Astrobase.

Follow the [simple-gcp](./simple-gcp#create-gke-cluster) and [simple-aws](./simple-aws#create-eks-cluster) instructions for creating clusters only, then run the following command to deploy services across both clouds

```sh
$ astrobase apply -f resources.yaml -v "PROJECT_ID=$(gcloud config get-value project) EKS_LOCATION=us-east-1 GKE_LOCATION=us-central1-c NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
```
