# Scheduling

## Manual Scheduling

You can manually schedule pods using the `nodeName` field in the configuration file.

The built-in scheduler goes through all the pods and looks at which pods have this field set.

If there is no `nodeName` the status will be pending.

You can only specify `nodeName` at creation process. Therefore, you can mimic what the scheduler does by creating a binding file and posting it to the pod.

## Labels and Selectors

You can group and select objects using labels and selectors

You can create as many custom labels in the YAML definition files.

E.g. to get pods with a specific label:

kubectl get pods --selector label=label-name

Annotations can be used to record other additional details for information such as build or version information.

## Taints and Tolerations

Taints and tolerations are used to set restrictions on what pods can be scheduled on a node.

We can place a taint on a node, and by default, pods do not have any tolerations, therefore they cannot go to that node with the taint.

To create a taint on a node:

kubectl taint nodes <node-name> <key>=<value>:<taint-effect>

There are three `taint-effects`:

- `NoSchedule` - Pods will not be scheduled on the node.
- `PreferNoSchedule` - It will try not to schedule pod onto node but not guaranteed.
- `NoExecute` - New pods will not be scheduled and existing pods (if any) will be evicted.

All values for taints and tolerations in the YAML file, need to have double quotes.

## Node Affinity

Provides advanced capabilities to limit pod placement on specific nodes.

An example of the affinity section of a YAML file is shown below:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Small
```

There are numerous operators such as `NotIn` and `Exists` and node affinity types such as:

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`
- `requiredDuringSchedulingRequiredDuringExecution`

Node Affinity and Taints/Tolerations should be used together to completely dedicate nodes to specific pods.

## Resource Requirements and Limits

Kube scheduler determines which pod goes to a node, it determines which is the best node for the pod to go to based on resources such as CPU and memory. If there is insufficient node resources then the pod will be in a pending state.

You can specify resources specification in the pod definition YAML file to tell the scheduler to look for a node with that minimum requirements to place the pod into. Also, you can specify limits to limit the containers resource usage.

```yaml
spec:
  containers:
  - name: example
    resources:
      requests:
        memory: "1Gi"
        cpu: 1
      limits:
        memory: "2Gi"
        cpu: 2
```

If a pod exceeds the CPU limit, the system throttles the CPU so it does not go over the limit, it cannot use more than the limit. For memory, a pod can use more memory than its limit but if it does this constantly, it will be terminated with a OOM (Out of Memory) error.

### Limit Range

Limit Ranges can set default values to pods without any requests or limits, these are set at the namespace level:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-cpu-constraint
spec:
  limits:
  - default:
      cpu: 500m
    defaultRequest:
      cpu: 500m
    max:
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```

### Resource Quota

Restrict total amount of resources in a kubernetes cluster. All the combined pods should not exceed a specified resource amount. Resource Quotas limit the resources across all pods together. These are applied at a namespace level:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example-quota
spec:
  hard:
    requests.cpu: 4
    requests.memory: 4Gi
    limits.cpu: 10
    limits.memory: 10Gi
```

## Daemon Sets

Daemon sets are similar to replica sets to help deploy multiple instances of pods. It runs one copy of a pod on each node in the cluster. Whenever a new node is added to the cluster, a replica of the pod is automatically added to that node. This ensures one copy of the pod is always present in all nodes in the cluster.

It can be used for monitoring and logging of nodes. It can be used for kube-proxy pods inside each nodes. Also, it can be used for networking inside each node such as weave-net.

```yaml
apiVersion: apps/v1
kind: DaemonSet
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

## Static Pods

Kubelets can manage a node independently, without the master or API server. The pod definition file can be provided to the kubelet by having it read from the `/etc/kubenetes/manifest` folder. Therefore, the kubelet has visibility of this and can automatically restart or bring up the pod.

The kubelet needs a setting changed in its service so it can read from this folder, for example: `--pod-manifest-path=/etc/kubernetes/manifests` or `--config=kubeconfig.yaml` and create a yaml file to define the `staticPodPath`.

These pods that are created this way are called static pods, and only applies to pods not services etc.

We cannot use the `kubectl` commands for static pods, so we have to use `docker` commands.

## Multiple Schedulers

You can create multiple schedulers by creating a custom one in addition to the default scheduler. Schedulers can be created using a configuration YAML file as below:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: example-scheduler
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: lock-object-my-scheduler
```

### Deploy Scheduler as a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --config=/etc/kubernetes/config.yaml
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
    name: kube-scheduler
```
