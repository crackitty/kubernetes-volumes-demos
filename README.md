# Kubernetes Volumes Demos

Simple Mongo deployment

## Install MongoDB and expose it via a NodePort Service

```bash
kubectl apply -f mongodb/deploy.yaml
kubectl apply -f mongodb/service.yaml
```
