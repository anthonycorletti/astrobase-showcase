# Argo on GCP

Workflows, Continuous Delivery, Rollouts, and Events.

## Check your Astrobase CLI and Server

```sh
$ astrobase check commands
docker found at: /usr/local/bin/docker
gcloud found at: /usr/local/google-cloud-sdk/bin/gcloud
kubectl found at: /usr/local/bin/kubectl
$ astrobase version && curl -s -X GET "http://:8787/healthcheck" | jq
🚀 Astrobase CLI 0.1.6 🧑‍🚀
{
  "api_version": "v0",
  "api_release_version": "0.1.3",
  "message": "We're on the air.",
  "time": "2021-04-09 16:16:46.192557"
}
```

## Create your cluster

```sh
$ astrobase apply -f cluster.yaml -v "PROJECT_ID=$(gcloud config get-value project) LOCATION=us-central1-c"
{
  "name": "operation-1617985580485-41424f47",
  "zone": "us-central1-c",
  "operationType": "CREATE_CLUSTER",
  "status": "RUNNING",
  "selfLink": "https://container.googleapis.com/v1beta1/projects/715209933323/zones/us-central1-c/operations/operation-1617985580485-41424f47",
  "targetLink": "https://container.googleapis.com/v1beta1/projects/715209933323/zones/us-central1-c/clusters/workflows",
  "startTime": "2021-04-09T16:26:20.485875815Z"
}
```

## Apply Argo Workflow Resources

```sh
$ astrobase apply -f resources.yaml
```

Check that Argo is running

```sh
$ kubectl port-forward svc/argo-server 2746:2746
```

Then visit https://:2746 in your browser. There will be a self signed cert that you can safely bypass.

You will need to add an admin rolebinding for this example

```sh
$ kubectl create clusterrolebinding default-admin-argo --clusterrole=cluster-admin --user=system:serviceaccount:default:default
```

## Run a workflow!

```sh
$ argo submit --watch workflows/workflows/hello-world.yaml
```
