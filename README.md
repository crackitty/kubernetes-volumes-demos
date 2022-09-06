# Kubernetes Volumes Demos

## Creating the local cluster with Kind

Run

```bash
kind create cluster --config
```

Simple Mongo deployment

## Install MongoDB and expose it via a NodePort Service

```bash
kubectl apply -f mongodb/deploy.yaml
kubectl apply -f mongodb/service.yaml
```

## Volumes

Before we showed that we can add data to our application, but if we kill the container
then we lose the data.

Now we will show that by adding a `Volume` of type `emptyDir`, the data is stored at the
Pod level so if the container restarts, we will still have our data.

So, this way, the data is available for as long as the Pod is alive.
