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

> Note: The data is being stored on the Node where the Pod is running

We can SSH into the Node - we're running Kind here, so we need to connect using docker

```bash
docker exec -it mongo-demo-worker sh
```

If you look in the folder

```bash
ls /var/lib/kubelet/pods
```

You'll see the hashes of all the pods that were running on the node. To find our Pod
we can run

```bash
kubectl get pod
```

and then run the following to find the full pod ID

```bash
kubectl get pod <pod-name> -o yaml
```

Or nicer:

```bash
kubectl get pod <pod-name> -o jsonpath='{.metadata.uid}'
```

Take the UID of the pod and then, in the shell you have opened into the Node, run:

```bash
ls /var/lib/kubelet/pods/<POD-UID>/volumes/kubernetes.io~empty-dir/mongo-volume
```

and you'll see the contents of the MongoDB data directory that was mounted

## Persisting Volumes across Pod restarts

The previous step allowed us to save data across container restarts, but not Pod
restarts. Next we will look at this case.

So, we can first delete the pod to show that the data is lost when we recreate
the Pod.

```bash
kubectl delete pod <pod-id>
```

We can see that a new Pod is created immediately by our Deployment because we have
said we want 1 replica to be guaranteed so Kubernetes spins up a new Pod for us.

We can list the contents of the same directory in the docker session for our node
and we see that the folder no longer exists.

The same thing can be seen if we connect to Mongo Compass

The data that was in this Pod, in our `emptyDir` is only available to the Pod, and
any other Pods on the same Node cannot ever access this data.

To fix this we can move the Volume out of the Pod and store it on the Machine where
the Node is running. For this we will use `hostPath`. It allows us to point out to
the hosts file-system and store our files there.

This is not something to be done in a production setup because you can probably see
that it might be a security concern if we allow Pods to write to the underlying
host file-system. But there are cases where this may be both necessary and useful.

There are quite a few CVE's (Common Vulnerabilities and Exposures) around HostPaths

Examples:

- You might need to deploy some Node specific files through a Pod
- You might want to run a container that needs to access some internal docker functionality

Apply the changed deployment YAML

```bash
kubectl apply -f mongodb/deploy.yaml
```

Now we can delete this pod by scaling down the deployment to 0 replicas

```bash
kubectl scale deployment mongo --replicas=0
```

And then scale it back up to 1 and see that the Pod ID has changed, but when you
port-forward to the service again and refresh the db, you see the data there.

This works fine when we are dealing with just one Node... but it's not enough.