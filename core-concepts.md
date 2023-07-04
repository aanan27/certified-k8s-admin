# Core Concepts

## Docker vs ContainerD

| CLI Tool | `ctr` | `nerdctl` | `crictl` |
| -------- | --- | ------- | ------ |
| Purpose  | Debugging | General | Debugging |
| Community  | Containerd | Containerd | Kubernetes |
| Works with | Containerd | Containerd | All CRI Runtimes |

## ETCD

ETCDCTL is the CLI tool used to interact with ETCD.

ETCDCTL can interact with ETCD Server using 2 API versions - Version 2 and Version 3.  By default its set to use Version 2. Each version has different sets of commands.

Runs ETCD which listens on port 2379

```bash
./etcd
```

Store a key-value pair

```bash
./etcdctl set key1 value1
```

Get a value from a key

```bash
./etcdctl get key1
```

Set version of the ETCD API and check

```bash
export ETCDCTL_API=3
./etcdctl version
```

To store a key-value pair in Version 3:

```bash
./etcdctl put key1 value1
```

To explore the etcd database

```bash
kubectl get pods -n kube-system
```

To list all keys stored by Kubernetes, run the following command inside the etcd-master pod

```bash
kubectl exec etcd-master -n kube-system etcdctl get / --prefix -keys-only
```

## Kube-API Server

View Kube-API server options in an existing cluster if set up using kubeadm

```bash
kubectl get pods -n kube-system
```

In a non-kubeadm set up you can see the API options using the following:

```bash
cat /etc/systemd/system/kube-apiserver.service

ps -aux | grep kube-apiserver
```

## Kube Controller Manager

A controller is a process that continuously monitors the state of various components within the system nd works towards bringing the system to the desired state.

For example, the node-controller monitors the nodes using the kube-apiserver every 5 seconds to check the health of pods.

View Kube Controller Manager options using kubeadm namespace as previous or use the command in a non-kubeadm environment:

```bash
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

ps -aux | grep kube-controller-manager
```

## Kube Scheduler

Decides which pod goes where, it can filter nodes based on metrics such as CPU etc. and then it can rank nodes based on how much resources the node has.

View the scheduler in the kube-system namespace as previous or use the command:

```bash
cat /etc/kubernetes/manifests/kube-scheduler.yaml

ps -aux | grep kube-scheduler
```

## Kubelet

Point of contact between other worker nodes and the master. Kubelets registers nodes, creates pods and monitors nodes and pods.

> Note: Kubeadm does not deploy Kubelets

```bash
ps -aux | grep kubelet
```

## Kube Proxy

Every pod can communicate with every other pod using an internal pod network over the cluster. A service is created so the pods can be accessed, this also gets an IP address assigned to it. Whenever a pod tries to access a service using its IP or name, it forwards the traffic to the backend pod.

Kube-proxy runs on each node which looks for new services, it then creates the appropriate rules so it can connect with the new services using IP rules tables.

```bash
kubectl get daemonset -n kube-system
```

## Kube Pods

Pods are single instances of an application, one container instance per pod, they the smallest object you can create in Kubernetes.

You can deploy multiple pods inside a node or launch a new node and pod.

A single pod can have multiple container instances only if they are different applications such as having a 'helper' container.

Create a pod using the command:

```bash
kubectl run <application> --image <container-image>
```

Get more details about a specific pod

```bash
kubectl get pods

kubectl describe pod <pod-name>
```

Alternatively, write a YAML file to specify the desired pod and use the following command to create from file:

```bash
kubectl create -f <file-name.yaml>
```

```bash
kubectl run --image=nginx nginx - o yaml --dry-run=client
```

## Replica Sets

Replicas allow for high availability where you state a number of pods to run, and then the replication controller can bring up new pods to meet the replica set if any go down.

### Replication Controller

A replication controller can control pods over multiple nodes.

Replication Controller is being replaced by a Replica Set.

In a YAML file we can define a `ReplicationController` (RC) using a template spec. We then can define the number of `replicas` needed.

### Replica Set

In a YAML file, you need to specify `apps/v1` version for replica sets to work.

It also uses a template spec. and replicas, but also needs a `selector` which can match labels to the pods so the replica set only monitors the pods with that specific label meaning it only will bring up pods to meet the desired replicas within that pod replica set.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
```

We can update the replicas number in the YAML file to scale up the replicas of pods and run either of the following commands to update the file:

```bash
kubectl edit rs <rs-name>

kubectl replace -f <file-name.yaml>
```

Or we can use the scale command so we do not have to update the YAML file:

```bash
kubectl scale --replicas=6 -f <file-name.yaml>

kubectl scale --replicas=6 replicaset <replicaset-name>
```

## Deployments

Can deploy the application in multiple instances in different environments such as production, development etc. You can update changes or rollback in certain instances.

Deployments are one level higher and can use replica sets and also control updates and changes.

To get information about all objects, use the following command:

```bash
kubectl get all
```

Use the following command to create a deployment:

```bash
kubectl create deployment <deployment-name> --image=<image-name>
```

Generate the deployment YAML file (use --dry-run to test and not create it)

```bash
kubectl create deployment <deployment-name> --image=<image-name> --replicas=<n> --dry-run=client -o yaml > <file-name.yaml>
```

## Services

Services enable connectivity between groups of pods, they enable frontend apps to be made available to end users, helps establish connection to backend apps and to external data sources etc.

Types of Services:

- `NodePort`: makes internal pod accessible on a port on the node.

  - `targetPort`: port on the pod that the service forwards to

  - `port`: port on the service itself

- `ClusterIP`: creates virtual IP address inside the cluster to enable communication between different services.

- `LoadBalancer`: provisions load balancer for our application in supported cloud providers.

## Namespaces

Namespaces help to isolate resources such as having a 'Dev' or 'Prod' namespace, for example, so you do not accidentally modify resources in Production.

Default, kube-system and kube-public namespaces are created by default.

In each Namespace, resources communicate with each other using their names such as 'db-service'. For a resource outside of the namespace to connect, it needs to append the name of the namespace e.g. 'db-service.dev.svc.cluster.local'

External namespace format: 'service-name.namespace.service.domain'

To create a pod inside a specific namespace use the command:

```bash
kubectl create -f <file.yaml> --namepsace=<namespace>
```

Or you can add `namespace` definition to the YAML file.

To create a new namespace, write a YAML file to define the namespace or create it using a command:

```bash
kubectl create -f <namespace-file.yaml>

kubectl create namespace <name>
```

To switch namespaces:

```bash
kubectl config set-context $(kubectl config current-context) --namespace=<desired-namespace>
```

Or to view pods in all namespaces:

```bash
kubectl get pods --all-namespaces
```

## Imperative vs Declarative

**Imperative:** giving instructions to reach a desired result, specifying how to get to the result.

- Instructions; What and How
- E.g. run, create, edit, scale, delete etc. commands

**Declarative:** you declare/specify the desired result and the system figures out what to do to reach it.

- Declaring end goal; Simple
- E.g. apply command, to look at existing configuration and figure out what changes need to be made

## Kubectl Apply Command

The `kubectl apply` command compares the local configuration file and the last applied configuration before making a decision on what changes are to be made.

The last applied configuration can be found as a JSON file:

'kubectl.kubernetes.io/last-applied-configuration'
