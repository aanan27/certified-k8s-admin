# Logging and Monitoring

## Monitoring

Kubernetes does not have a built-in logging and monitoring feature for Kubernetes clusters. You can use software such as Metrics Server, Prometheus, Elastic Stack, Datadog or Dynatrace to monitor metrics and provide analytics of Kubernetes resources.

Heapster was the original projects to enable monitoring and analysis features for Kubernetes. It has now been deprecated and cut down into Metrics Server. It uses metrics from each node and pod and stores the metrics in-memory. Therefore, you cannot see historical data unlike the previously mentioned software.

To enable Metrics Server use the following commands:

- for minikube clusters:

    ```bash
    minikube addons enable metrics-server
    ```

- for all others

    ``` bash
    git clone https://github.com/kubernetes-incubator/metrics-server.git

    kubectl create -f deploy/1.8+/
    ```

To see performance metrics of node and pods use the following commands:

```bash
kubectl top node

kubectl top pod
```

## Logging

You can view the logs of a resource such as a pod using the following `logs` command (`-f` to view live stream of logs):

```bash
kubectl logs -f <object-name>
```

If there are multiple containers in a pod, you need to explicitly specify the name of the container you want to view the logs of:

```bash
kubectl logs -f <object-name> <container-name>
```
