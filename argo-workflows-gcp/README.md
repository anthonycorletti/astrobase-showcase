# Argo on GCP

Workflows and Events.

## Check your Astrobase CLI and Server

```sh
$ astrobase check commands
docker found at: /usr/local/bin/docker
kubectl found at: /usr/local/bin/kubectl
$ astrobase version && curl -s -X GET "http://:8787/healthcheck" | jq
🚀 Astrobase CLI 0.2.1 🧑‍🚀
{
  "api_version": "v0",
  "api_release_version": "0.1.6",
  "message": "We're on the air.",
  "time": "2021-05-14 20:44:31.766744"
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

## Apply Argo Workflow and Events Resources

```sh
$ astrobase apply -f resources.yaml -v "PROJECT_ID=$(gcloud config get-value project) LOCATION=us-central1-c"
```

You may need to run that again incase the API server doesn't start in time to create the Argo Event, EventSource, and Sourcer api endpoints.

Check that Argo is running ...

```sh
$ kubectl port-forward svc/argo-server 2746:2746
```

Then visit https://localhost:2746 in your browser. There will be a self signed cert that you can safely bypass.

Copy your auth token with `argo auth token | pbcopy` and use that to log in.

## Run a workflow

```sh
$ argo submit --watch workflows/hello-world.yaml
```

## Events

Now that we have workflows running, let's trigger a workflow with a webhook.

```sh
$ kubectl port-forward $(kubectl get pod -l eventsource-name=webhook -o name) 12000:12000
$ curl -d '{"message": "hello world"}' -H "Content-Type: application/json" -X POST http://localhost:12000/example
success
```

And check that the workflow was emitted!

```sh
$ kubectl get workflows -l events.argoproj.io/sensor=webhook
NAME            STATUS      AGE
webhook-w66xz   Succeeded   60s
```

## Cleanup!

```sh
$ astrobase destroy -f cluster.yaml -v "PROJECT_ID=$(gcloud config get-value project) LOCATION=us-central1-c"
```

