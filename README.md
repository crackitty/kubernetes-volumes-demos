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

## Persistent Volumes

Up until now, we've been dealing with **Ephemeral Volumes** which means that if
the container, pod or node that holds the data gets deleted, then the volume is
deleted with it.

Now we have the more real world case where we have multiple Nodes (worker nodes)
that the Kubernetes scheduler will schedule Pods on at will and we may not know
which Node the Pod will end up on because it decides this based on available
resources. So now, imagine the case where we have the data stored on Node 1
using hostPath and our new Pod (replica 2 for example) gets scheduled on Node 2.
First of all, the Pod on Node 2 cannot access the data on Node 1, but even
worse, if Node 1 fails for some reason, then all our db data is lost.

So the solution is to move the storge out further, to the cloud, or an external
NFS disk etc. i.e. on a machine that is not a Node in your cluster. That way, failures
in the Nodes cannot result in the data being lost.

The tools we will have to do this are

- Persistent Volumes
- Persistent Volume Claims
- Storage Classes

We will look at each of these in turn as we go through a real world scenario.

We will create a Persistent Volume first to see how it works

We can get the API version of the Persistent Volume by running the following

```bash
kubectl api-resources | grep Persistent
```

The accessModes available are:

- ReadWriteMany
- ReadWriteOnce
- ReadOnlyOnce
- ReadOnlyMany
- ReadWriteOncePod

So now we can create this PV in the cluster

```bash
kubectl apply -f mongodb/pv.yaml
```

Now we have a PV created in our cluster so we have a way to store data
outside our nodes. How do we use this in our MongoDB application?

## Persistent Volume Claim

We use this to ask kubernetes can it match us with a PV that satisfies
our needs for **capacity of 5Gi** and **accessMode of RWX**. These are
the main two criteria Kubernetes uses in our example to match. There
can and are other criteria, but for now it's enough to consider these
as the matching criteria.

We can create the PVC now...

```bash
kubectl apply -f mongodb/pv.yaml
```

We also need to update the deployment to use this PVC. So we changed the
deployment to use the new PVC and we can apply that change

```bash
kubectl apply -f mongodb/deploy.yaml
```

If you check the Pods now you'll see that the new Pod hasn't
come up properly - so use `kubectl describe` to investigate.

Sometimes the old hostPath doesn't detach from the Pod and
we end up with phantom Pods, so to fix this, we can scale
down the replicas to 0 and back up to 1.

So, in this scenario, the Deployments are created by the
developers and the PV's would be created by some admin on
request of a developer and then the developer can create
a PVC which will match and bind to it.

But this is a bit of a hassle - we don't want to have to
sit waiting for requests for PV's. There must be a more
dynamic way to do this so develpers are free to work away
without waiting on a PV to be manually provisioned.

There are...

## Storage Classes

Storage classes allow us to use a Storage Provisioner to do
what a human would normally do on our behalf. i.e. create the
PV for us based on certain requirements.

Usually, though, this manual process is something we want to
avoid (tickets and the like).

So Storage classses allow us to have dynamic storage. Storage
Classes use provisioners to to the work for them. See the
documentation for the list of provioners. In our case, we
currently have only one storage class we can use, "tanzu-stp"
which is a CSI provisioner from vsphere. Unfortunately it's
not configured to allow using RWX so our demos can only
use RWO.

You can run these on a Minikube cluster and demo the functionality.
However, they also don't come with RWX available so you would
have to add something like an NFS storage class.