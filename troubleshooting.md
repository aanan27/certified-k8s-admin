# Troubleshooting

## Application Failure

1. Check the service and deployment names, to ensure that they can communicate with each other via their object name.
2. Check the port and target port numbers on the services.
3. Check the labels and selectors to ensure they are pointing to the correct label on the pods.
4. Check the user credentials are correct on the pods.
5. Check the environment variables are all correct for example, for the application to connect to the database.
6. Check the endpoints and the port number is correct for the app trying to access the port of the database.

## Control Plane Failure

1. Check the pods if it can be scheduled to a node, if not, check the Kube Scheduler, remembering it is a static pod so you can check the configuration at `/etc/kubernetes/manifest/`.
2. Scale pods using the `scale` command or by editing the number of `replicas` in the deployment configuration.
3. If the pods do not scale still, there may be an issue with the controller manager.
4. Check the controller manager for filepaths for certificate files etc.

## Worker Node Failure

1. Check condition of node using `describe node` command, restart Kubelet if needed.
2. Check Kubelet logs for possible issues `service kubelet status` and `sudo journalctl -u kubelet`. Check Kubelet config file at `/var/lib/kubelet/config.yaml`
3. Check the port number on the Kubelet at `etc/kubernetes/kubelet.conf` which by default is 6443.

## Troubleshoot Network

1. Check the nodes and pods status, if there is a networking issue allocating IP addresses etc. Check if there are any network containers deployed such as flannel or weave. For example, install weave and wait until the pos are created.

    ```bash
    kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
    ```

2. Check Kube-proxy and the configuration files by viewing the config map and editing the Kube proxy by editing it as a daemon set.
