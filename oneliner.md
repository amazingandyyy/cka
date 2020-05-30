# Kubectl One Liner (Imperative Commands with Kubectl)

- [documention](https://kubernetes.io/docs/reference/kubectl/conventions/)

```sh
# ===> Create a pod
$ kubectl run --generator=run-pod/v1 nginx --image=nginx
$ kubectl run --generator=run-pod/v1 nginx --image=nginx --dry-run -o yaml
$ kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml
# (name is broken)

# ===> Create deployment
$ kubectl create deployment --image=nginx nginx
$ kubectl create deployment --image=nginx nginx --dry-run -o yaml
# (no replicas options)
$ kubectl create deployment --image=nginx nginx --dry-run -o yaml > nginx-deployment.yaml
# (then change replicas)

# ===> Create services

# CASE 1
# Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379
$ kubectl expose pod redis --port=6379 --name redis-service --dry-run -o yaml
# (This will automatically use the pod's labels as selectors)
$ kubectl create service clusterip redis --tcp=6379:6379 --dry-run -o yaml
#  (This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service)

# CASE 2
# Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:
$ kubectl expose pod nginx --port=80 --name nginx-service --dry-run -o yaml
$ kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run -o yaml

# CASE 3
# Deploy a redis pod using the redis:alpine image with the labels set to tier=db.
$ kubectl run --generator=run-pod/v1 redis --image=redis:alpine -l tier=db

# CASE 4
# create a static pod
$ kubectl run --restart=Never --image=busybox static-busybox --dry-run -o yaml --command -- sleep 1000 > /etc/kubernetes/manifests/static-busybox.yaml
```

