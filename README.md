# Certified Kubernetes Administrator (CKA)

## Architecture

![img](https://i.imgur.com/dfSsQ5g.png)

![img](https://i.imgur.com/zoGkzsq.png)

- Master Node
  - ETCD cluster: store information about the cluster
  - Kube Controller Manager: take care of different functions, like node controller, replica controller...etc
  - kube-apiserver: responsible for orchestrating all opeations within the cluster
  - kube-scheduler: scheduling applications on nodes
- Worker Node
  - kubelet: listen to the instructions from the kube-apiserver
  - kube-proxy: enabling communication between services within the cluster
  - container runtime engine

## Core

### ETCD cluster

- ETCD is a distributed k-v store
- ETCD in K8
  - every change of nodes, pods, configs, secrets, accounts, roles, binding... will change the etcd, and only when the change of etcd is finished, it's considered to be completed
- Setups:
  - manual
  - kubeadm
    - `kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl get / --prefix --keys-only --limit=10 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt  --key /etc/kubernetes/pki/etcd/server.key"`
    - it will output a lot of registry
      - looks like `registry/pods/name...etc`

### kube-apiserver

- authenticate user, validate request, retrieve data
- the only component that update ETCD, and interact with schedule and kubelet
- view api-server options
  - locate in kube-apiserver-minikube pods' `/etc/systemd/system/kube-apiserver.service`

### Kube controller manager

- controller examples
  - watch status, remediate situation
- a lot of controllers
  - node-controller: node monitor period = 5s, node monitor grace period = 40s, pod eviction timeout = 5m
  - replication-controller: make sure the replicas is enough
  - many other controllers, deployment-controller, namespace-controller, endpoints-controller, job-controller, replication-controller, replicaset, node-controller
- view options
  - locate in kube-controller-manager-master pods' `/etc/kubernetes/manifests/kube-controller-manager.yaml` or (non-kube-admin-setup) `/etc/systemd/system/kube-controller-manager.service`

### Scheduler

- determine which pods should go to which node
- jobs: 
  - filter nodes
  - rank nodes
  - resource requirements/limits, taints/tolerations, node selectors/affinity
- view options
  - locate in kube-controller-manager-master pods' `/etc/kubernetes/manifests/kube-scheduler.yaml` or (non-kube-admin-setup) `/etc/systemd/system/kube-scheduler.service`

### kubelet

- captain of the ship, register nodes, create pods, monitor node/pods
- kubeadm does not deploy kubelets, you have to download and manually run it as a service
- view options
  - located in `/etc/systemd/system/kubelets.service`

### kube proxy

- match sevice name and the ip
- it's a process that runs on each node

### Pods

- pods is the smallest object you can create in k8, it's a single process of running the docker image
- there are also multi-container pod, one pods has multiple containers(helper containers, and each pods can refer to each other with `localhost`)

- example

```yml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: nginx-container
    image: nginx
```

- notes

| Kind | Version |
| ---- | --- |
| Pod | v1 |
| Service | v1 |
| ReplicaSet | apps/v1 |
| DaemonSet | apps/v1 |
| Deployment | apps/v1 |
| ConfigMap | v1 |
| Secret | v1 |

### ReplicaSet (Controller)

- example

```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rc
  labels:
    app: myapp
    tier: frontend
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        tier: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      tier: frontend
```

- to scale up
  - 1. change the yaml and `kubectl replace -f rc.yml`
  - 2. `kubectl scale --replicas=6 -f rc.yml`
  - 3. `kubectl scale --replicas=6 replicaset myapp-rc`

### Deployment

- upgrade/rollback pods, advance usage of other things
- create deployment will create rs

![img](https://i.imgur.com/dO49pGy.png)

### Namespceas

- namespcae can be specified in yaml's metadata or using `-n flag of kubectl
- using namespace yaml file or use `kubectl create namespace name`
- switch
  - kubectl config set-context $(kubectl config current-context) --namespace=dev

![img](https://i.imgur.com/bFGvOQ8.png)

### Service

| Service Types | Feature |
| :---- | :--- |
| NodePort | listen to node's port and then redirect to a pod's ip |
| ClusterIP | pods to pods communication |
| LoadBalancer | provision a LB on the cloud provider, then distribute traffic to pods |

#### NodePort

- config port, targetPort, and nodePort
![img](https://i.imgur.com/BzxxcKP.png)
  - then use selector to select the pods(random+affinity)
  ![img](https://i.imgur.com/R4rDAnl.png)
  - the service is smart enough to work with single pods in single node, multiple pods in single node, or multiple pods in multiple nodes, as long as it matches the selector conditions

#### ClusterIP

![img](https://i.imgur.com/XWNJxXj.png)

### Scheduling

- schedule scan all nodes and find one node best for the new created pod, then assign a name to the node and then add a nodeName to pod's yaml file. e.g. `spec.nodeName: node02`
- if there is not schedule and the pods' yaml doesn't have `spec.nodeName` set, then the pod status will be Pending state, manually create a Binding yaml file then send a POST request to `http://$SERVERapi/v1/namespaces/default/pods/$PODNAME/binding/` with the yaml converted JSON as body

### Label & Selector

- deployment/rs's selector uses matchLabels to match pods with label
- use selector as filter: `kubectl get pods --selector env=dev,tier=frontend`

### Taints & Tolerations

- taints on node, tolerations defined on pods
- commands: `kubectl taint node node01 key=value:taint-effect
  - taint-effect can be `NoSchedule(don't schedule new pods) | PreferNoSchedule(not guarantee) | NoExecute(will evict existing pods) `
  - e.g. `kubectl taint node node01 app=blue:NoSchdule`
  - the pods's tolerations then has to be
    ```yml
    spec.tolerations:
      - key: app
        operator: Equal
        value: blue
        effect: NoSchedule
    ```
  - it doesn't guarantee the certain pods will must go to certain node. (that will be achieved by affinity)

### Node Selector(Affinity)

- select a node
  ```yml
    kind: Pod
    ...
    spec.nodeSelector:
      size: Large
  ```
- label a node
  - `kubectl label nodes node01 size=Large`
- select a node with affinity
  ```yml
    kind: Pod
    ...
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoreDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: size
                operator: In
                values:
                - Large
                - Medium
              - key: size
                operator: NotIn
                values:
                - Small
              - key: good
                operator: Exists
  ```
    - requiredDuringSchedulingIgnoreDuringExecution or preferredDuringSchedulingIgnoreDuringExecution or preferredDuringSchedulingRequiredDuringExecution

### Resource requirements and limits

- container level of request and limits

```yml
spec.containers.resources:
  requests:
    memory: "1Gi"
    cpu: 1
```

- default requirement: 0.5CPU and 256Mi memory if the ns has that been set: ([docs](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/))
```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
      memory: 512Mi
    defaultRequest:
      cpu: 0.5
      memory: 256Mi
    type: Container
```

### DaemonSets

- Make sure the pods is running on every node, it use affinity under the hood to make sure the pods is running on every node

- example

  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: fluentd-elasticsearch
    namespace: kube-system
  spec:
    selector:
      matchLabels:
        name: fluentd-elasticsearch
    template:
      metadata:
        labels:
          name: fluentd-elasticsearch
      spec:
        containers:
        - name: fluentd-elasticsearch
          image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
  ```

### Static PODs

- only kubelet to create the pods, with yaml files to be put inside `/etc/kubernetes/manifests` folder.
- config
  - 1. pass `--pod-manifest-path=/etc/kubernetes/manifests` to kubelet.service
  - 2. pass `--config=kubeconfig.yaml` then inside `kubeconfig.yaml`, it is `staticPodPath: /etc/kubernetes/manifests`

![img](https://i.imgur.com/LJAZnGA.png)
![img](https://i.imgur.com/xsoXRbY.png)


- use `ps -aux | grep .yaml ` to find the config path, then cat that file `grep staticPodPath` to find the folder of those yaml files.

### Scheduling

- create extra schedulers

```yaml
...
spec:
 containers:
  - command:
    - kybe-scheduler
    - --leader-elect=false
    - --port=
    - --scheduler-name=my-scheduler
    - --lock-object-name=my-scheduler
...
```
  
  - then add `spec.schedulerName`

- steps to find where to add pods
  ```sh
  $ cd /etc/systemd/system/kubelet.service.d
  $ cat 10-kubeadm.conf | grep kubelet
  $ cat /var/lib/kubelet/config.yaml | grep -i path
  # copy and config the file
  $ netstat -natulp | grep 10253 # until finding a port that is not being used
  # then add `spec.schedulerName` to pods' yaml

  ```

### Monitoring/Logging

- metrics
  - enable it: `minikube addons enable metrics-server`
  - then can use `kubectl top pods` or `kubectl top nodes`
- logging
  - `kubectl logs -f pod-name container-name`

## Application Lifecycle Management

### Rollout and versioning

- `kubectl rollout status deployment/myapp-deployment`
- `kubectl rollout history deployment/myapp-deployment`
- deployment strategy
  - Recreate
  - RollingUpdate(default)
- what counts as upgrade
  - 1. update yaml file
  - 2. `kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1`
- rollback
  - `kybectl rollout undo deployment/myapp-deployment`
- kubectl
  ```sh
  kubectl create -f myapp-deployment.yml
  kubectl get deployment
  # ugprade
  kubectl apply -f myapp-deployment.yml
  kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
  # status
  kubectl rollout status deployment/myapp-deployment
  kubectl rollout history deployment/myapp-deployment
  kubectl rollout undo deployment/myapp-dep1
  ```

### Config applications

- config commands/args, environment variables, secrets
- commands/args
  - command in k8.yaml is the ENTRYPOINT in dockerfile
  - args in k8.yaml is the CMD in dockerfile
- environment variables
  ```yml
  env
   - name: APP_COLOR
     value: pink
  ---
  env
   - name: APP_COLOR
     valueFrom:
       configMapKeyRef:
  ---
  env
   - name: APP_COLOR
     value:
        secretKeyRef:
  ```

### ConfigMap

- configuration data in kubernetes
- imperative:
  - 1. From literal: `kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod`
  - 2. From file: `kubectl create configmap app-config --from-file=app_config.properties`
- create:

  ```yml
  apiVersion: v1
  kind: ConfigMap
  metadate:
    name: app-config
  data:
    APP_COLOR: blue
    APP_MODE: prod
  ```

- inject into pod

  ```yml
  --- env
  envFrom:
  - configMapRef:
      name: app-config
  --- single env
  env
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR
  --- volume
  volumes:
    - name: app-config-volume
      configMap:
        name: app-config
  ```

### Secret

- configuration secret in kubernetes
- imperative:
  - 1. From literal: `kubectl create secret generic secret-name --from-literal=DB_User=root --from-literal=DB_Password=passwooord`
  - 2. From file: `kubectl create secret generic secret-name --from-file=app_secret.properties`
- create:
  - do `echo -n passwooord | base64` then add them to 
  - do `echo -n cGFzc3dvb29yZA== | base64 --decode` to decode
  - config Secret yaml

    ```yml
    apiVersion: v1
    kind: Secret
    metadate:
      name: app-secret
    data:
      DB_User: cm9vdA==
      DB_Password: cGFzc3dvb29yZA==
    ```

- inject into pod

  ```yml
  --- env
  envFrom:
    - secretRef:
        name: secret-name
  --- single env
  env
  - name: DB_Password
    valueFrom:
      secretKeyRef:
        name: secret-name
        key: DB_Password
  --- volume
  volumes:
    - name: app-config-volume
      secret:
        secretName: app-config # (which will create files such as /opt/app-config-volumes/DB_Password and /opt/app-config-volumes/DB_Host)
  ```

### Multi-container Pods

- share the same lifecycle, network, and storage, create together and destroy together
- initContainers([docs](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/))

  ```yml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
  spec:
    containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    initContainers:
    - name: init-myservice
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
    - name: init-mydb
      image: busybox:1.28
      command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
  ```

## Cluster Maintenance

- `kubectl drain node-1` // will move pods inside to other nodes and cordon the node
- `kubectl uncordon node-1` // unstrict(uncordon) scheduling
- `kubectl cordon node-2` // if the node is running a critical service

### Cluster Upgrade Process

![img](https://i.imgur.com/nB7oZOc.png)

- Choises
  - 1. few clicks on GCP
  - 2. use kubeadm
  - 3. manual
- strategies
  - upgrade master first then worker node
  - upgrade worker node:
    - 1. bring down all then upgrade all
    - 2. rolling upgrading
    - 3. add new nodes of new version, then move pods
- upgrade one cluster process

  ```sh
  $ kubectl drain node01
  # plan on upgrading cluster
  $ kubeadm upgrade plan
  # install kubeadm
  $ apt install kubeadm=1.17.0-00
  # upgrade kubeadm
  $ kubeadm upgrade apply v1.17.0
  # install kubelet
  $ apt upgrade kubelet=1.17.0-00
  # upgrade kubelet
  $ kubeadm upgrade node config --kubelet-version v1.17.0
  $ systemctl restart kubelet
  $ kubectl uncordon node01
  ```

- Case study
  - `kubeadm upgrade node` on node is like `kubeadm upgrade apply` on master
  - to upgrade the master node, run

    ```sh
      kubectl drain master --ignore-daemonsets
      apt install kubeadm=<version>
      kubeadm upgrade apply <version>
      apt install kubelet=<version>
      kubectl uncordon master
    ```

  - to upgrade the worker nodes, run
  
  ```sh
    kubectl drain node01 --ignore-daemonsets
    ssh node01
    node01$ apt install kubeadm=<version>
    node01$ apt install kubelet=<version>
    node01$ apt-mark hold kubelet
    master$ kubectl uncordon node01
  ```


- use documention [no matter what](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

### Backup & Restore

- backup configuration
  - `kubectl get all --all-namespaces -o yaml > all-deploy-backup.yaml`
- ETCD cluster ([docs](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster))
  - `etcdctl snapshot save snapshot.db`
  - [backup](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/practice-questions-answers/cluster-maintenance/backup-etcd/etcd-backup-and-restore.md#2-backup)

    ```sh
    ETCDCTL_API=3 etcdctl \
      --endpoints=https://[127.0.0.1]:2379 \ (--listen-client-urls)
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \ (--key-file)
      --cert=/etc/kubernetes/pki/etcd/server.crt \ (--cert-file)
      --key=/etc/kubernetes/pki/etcd/server.key \ (--peer-trusted-ca-file)
      snapshot save /tmp/snapshot-pre-boot.db
    ```
  - [restore](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/practice-questions-answers/cluster-maintenance/backup-etcd/etcd-backup-and-restore.md#3-restore-etcd-snapshot-to-a-new-folder) and update `/etc/kubernetes/manifests/etcd.yaml`

    ```sh
    ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --name=master \
     --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
     --data-dir /var/lib/etcd-from-backup \
     --initial-cluster=master=https://127.0.0.1:2380 \
     --initial-cluster-token etcd-cluster-1 \
     --initial-advertise-peer-urls=https://127.0.0.1:2380 \
     snapshot restore /tmp/snapshot-pre-boot.db
    ```

## Security

### Basic Authentication

- 1. create user list in `/tmp/users/user-details.csv`

```csv
# User File Contents
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
password123,user4,u0004
password123,user5,u0005
```

- 2. link with kube-apiserver in `/etc/kubernetes/manifests/kube-apiserver.yaml` with `--authorization-mode` and `--basic-auth-file` flags

```yml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
      ...
    - --basic-auth-file=/tmp/users/user-details.csv
```

- 3. create `kind: Role` and `kind: RoleBinding` yaml files

```yml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

### TLS

- get all list by get details
- decode with `openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout`

### Certificate API

- create CSR
  - 1. `openssl genrsa -out jane.key 2048`
  - 2. `openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr`
  - 3. `cat jane.csr | base64 | tr -d '\n'`
  - 4. create CertificateSigningRequest

  ```yml
  apiVersion: certificates.k8s.io/v1beta1
  kind: CertificateSigningRequest
  metadata:
    name: jane
  spec:
    groups:
    - system:authenticated
    usages:
    - digital signature
    - key encipherment
    - server auth
    request:
      GLKHLJKYHIHFVIFH...YUGHJGJKLHK
  ````

- recreate

```sh
openssl x509 -req -in /etc/kubernetes/pki/apiserver-etcd-client.csr -CA /etc/kubernetes/pki/etcd/ca.crt -CAkey /etc/kubernetes/pki/etcd/ca.key -CAcreateserial -out /etc/kubernetes/pki/apiserver-etcd-client.crtmaster
```

### KubeConfig

- fsa

- admin can do
  - `kubectl get csr`
  - `kubectl certificate approve jane`

## CKAD

- health status check
  - livenessProbe
  - readinessProbe
- multi-container patterns
  - sidecar
  - adapter
  - ambassador
