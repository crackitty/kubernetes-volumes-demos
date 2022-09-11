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

## Step 1

We'll show now that by restarting the container, we lose our data.

We can exec into our Pod

```bash
k exec -it mongo-5b9fff8dc8-vbk5f -- bash
```

And then kill the `mongodb` process by running `kill 1`

this kicks us out and we can then check `k get pods` and see that it's not a new Pod, but there
was 1 container restart.

Port forward again to Mongo and check if the data is there... it's GONE!

Go back to the slides

## Volumes

Before we showed that we can add data to our application, but if we kill the container
then we lose the data.

Now we will show that by adding a `Volume` of type `emptyDir`, the data is stored at the
Pod level so if the container restarts, we will still have our data.

So, this way, the data is available for as long as the Pod is alive.

Apply the changed deployment

```yaml
      volumes:
        - name: mongo-volume
          emptyDir: {}
```

By running

```bash
kubectl apply -f mongodb/deploy.yaml
```

After doing this we can demo the same process of 

- `port-forwarding`
- connecting `Mongo Compass` and adding data
- exec into the pod
- kill the mongodb process
- see the new container created
- check if `Mongo Compass` still shows it

> Note: The data is being stored on the Node where the Pod is running

> Note also: if we delete the Pod, then the data is gone - since the emptyDir
> volume is owned by the Pod and is removed when the Pod is removed.

## If using KIND

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