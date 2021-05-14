# showcase/simple-azure

Deploying an Azure AKS Cluster and a few sample kubernetes resources to get started.

#### Configure your Azure Resource Group

```sh
$ az group create --name my-resource-group --location eastus
```

#### Create an Azure Active Directory Application and Register the Application to manage resources

Follow these links (assuming you have an admin owner configured as yourself already)

1. Register an application with Azure AD and create a service principal - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#app-registration-app-objects-and-service-principals
1. Assign a role to the application - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#assign-a-role-to-the-application
1. Get the tenant and App ID values for sign in - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#get-tenant-and-app-id-values-for-signing-in
1. Authentication via Applciation Secret - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#option-2-create-a-new-application-secret
1. Configure Access Policies for resources - https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#configure-access-policies-on-resources

#### Export Azure Credentials in the shell session where you initialize astrobase

```sh
export AZURE_SUBSCRIPTION_ID=<AZURE_SUBSCRIPTION_ID>
export AZURE_CLIENT_ID=<AZURE_CLIENT_ID>
export AZURE_CLIENT_SECRET=<AZURE_CLIENT_SECRET>
export AZURE_TENANT_ID=<AZURE_TENANT_ID>
```

#### Check Commands

```sh
$ astrobase check commands
docker found at: /usr/local/bin/docker
kubectl found at: /usr/local/bin/kubectl
```

#### Initialize Astrobase

```sh
$ astrobase init
Initializing Astrobase ...
Starting Astrobase server ...
Astrobase initialized and running at http://localhost:8787
```

#### Check your versions

```sh
$ astrobase version; curl -s http://:8787/healthcheck | jq
🚀 Astrobase CLI 0.2.1 🧑‍🚀
{
  "api_version": "v0",
  "api_release_version": "0.1.6",
  "message": "We're on the air.",
  "time": "2021-05-14 20:43:04.924411"
}
```

#### Create Your AKS Cluster

```sh
$ astrobase apply -f tests/assets/test-aks-cluster.yaml -v "RESOURCE_GROUP_NAME=my_resource_group_name"
{
  "message": "AKS create request submitted for my_resource_group_name"
}
```

Check the status of the cluster until it shows `"Succeeded"`

```sh
$ curl -s -X GET "http://localhost:8787/aks/astrobase-test-aks?resource_group_name=my_resource_group_name" | jq .provisioning_state
"Succeeded"
```

#### Deploy Resources

```sh
$ astrobase apply -f tests/assets/test-resources-aks.yaml -v "RESOURCE_GROUP_NAME=my_resource_group_name NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
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

#### Destroy Resources


```sh
$ astrobase destroy -f tests/assets/test-resources-aks.yaml -v "RESOURCE_GROUP_NAME=my_resource_group_name NGINX_CONTAINER_PORT=80 NGINX_MEM_REQUEST=64Mi NGINX_CPU_REQUEST=250m NGINX_MEM_LIMIT=128Mi NGINX_CPU_LIMIT=500m"
destroying resources in astrobase-test-aks@eastus
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
destroying resources in astrobase-test-aks@eastus
deployment.apps "nginx-deployment" deleted
```


#### Destroy Cluster

```sh
$ astrobase destroy -f tests/assets/test-aks-cluster.yaml -v "RESOURCE_GROUP_NAME=my_resource_group_name"
{
  "message": "AKS delete request submitted for astrobase-test-aks"
}
```

```sh
$ curl -s -X GET "http://localhost:8787/aks/astrobase-test-aks?resource_group_name=my_resource_group_name" | jq .provisioning_state
"Deleting"
```
