# Cluster Maintenance

## OS Upgrades

The master node waits for a period of 5 minutes by default before considering the node as dead, otherwise it will automatically bring up pods inside the node if it has a replica set.

If you know a node will be back online due to maintenance upgrades, you can purposefully `drain` the node of all pods, bringing them down and recreating them on another node. You can `cordon` the node to restrict any pods being scheduled on it until you `uncordon` it after the period of maintenance.

```bash
kubectl drain <node-name>

kubectl cordon <node-name>

kubectl uncordon <node-name>
```

## Cluster Upgrade

Version standard format:

v1.11.3 - major, minor, patch

To upgrade a cluster, you can use `kubeadm` as follows:

```bash
kubeadm upgrade plan

kubeadm upgrade apply
```

To upgrade the `kubeadm` and each node, for example, to 1.27.0. First, `drain` and `upgrade` the master (controlplane) node using the following commands, then `uncordon` it and using the same commands, `drain` and `upgrade` the worker nodes:

```bash
kubectl drain <node-name>

apt-get update

apt-get upgrade kubeadm

apt-get install kubelet=1.27.0-00

sudo systemctl restart kubelet

kubectl uncordon <node-name>
```

## Backup and Restore Methods

All cluster related information is stored in the ETCD including resource configuration files (declarative approach). You can backup these configuration files to a SCM service such as GitHub.

To backup resources created using an imperative approach, we can save them all as a configuration file using the API server and the following command:

```bash
kubectl get all -a-ll-namespaces -o yaml > all-deploy-services.yaml
```

Tools such as Velero can do this for you and help backup kubernetes cluster using the API.

The ETCD configuration is stored in the data directory (`--data-dir=/var/lib/etcd`), which can be configured to be backed up by a backup tool. ETCD also has a built-in snapshot tool using the following command to save a snapshot of the ETCD database:

```bash
ETCDCTL_API=3 etcdctl snapshot save --endpoints=127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot.db

ETCDCTL_API=3 etcdctl snapshot status snapshot.db
```

To restore the cluster from this backup:

```bash
service kube-apiserver stop

ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup
```

The ETCD configuration file (`/etc/kubernetes/manifests/etcd.yaml`) can now be configured to use this new data directory. Then, reload the service daemon and restart the ETCD adn API server.

```bash
systemctl daemon-reload

service etcd restart

service kube-apiserver start
```
