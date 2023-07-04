# Storage

All files related to Docker are stored in `/var/lib/docker`. It creates images and containers using layered architecture from the Dockerfile, for example:

1. Base Ubuntu layer
2. Changes in apt packages
3. Changes in pip packages
4. Source code
5. Update entrypoint

Docker can skip layers by reusing any layers from other previously built images such as those with a base Ubuntu layer.

**Storage Drivers** help manage storage on images and containers such as AUFS, ZFS, BTRFS, Device Mapper and Overlay. To persist storage, you need to create volumes which are handled by **Volume Drivers** plugins such as Local, Azure File Storage, Convoy and DigitalOcean Block Storage etc. for example, to provision volume from AWS EBS using RexRay Volume Driver:

```bash
docker run -it --name mysql --volume-driver rexray/ebs --mount src=ebs-vol,target=/var/lib/mysql mysql
```

Kubernetes uses **Container Runtime Interface (CRI)** to work with Container runtimes such as Docker, cri-o and rkt. It uses **Container Network Interface (CNI)** to work with networking vendors such as weaveworks and flannel. Also, Kubernetes uses **Container Storage Interface (CSI)** to work with storage solutions such as portworx, AWS EBS, Azure Disk etc.

These are universal standards to follow, so Kubernetes can use RPCs to create, delete and publish volumes.

## Persistent Volumes and Persistent Volume Claims

Docker containers are transient, they only last for the period of time when needed. To persist data processed by the containers, we create **Volumes**. Similarly in Kubernetes, we attach a volume to a pod. When the pod gets deleted, the volume remains on the host.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

This works fine for a single node, but for a multi-node cluster, pods will be use the same directory on all nodes and expect them to all have the same data yet they are not the same since they are on different servers.

Therefore, a **Persistent Volume (PV)** can be used to create cluster-wide pool of volumes to be used by users deploying applications on different nodes on the cluster by selecting storage from this pool using a **Persistent Volume Claim (PVC)**.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
  - ReadOnlyMany
  # ReadWriteOnce
  # ReadWriteMany
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/data
```

Every PVC is bound to a PV, Kubernetes will try to find sufficient storage for the claim (matching the access modes and storage capacity). You can also use labels and selectors to bind to the volume. Although, there may be times where a claim gets bound to a different capacity PV (as shown below). If there are no available PVs, the PVC will be in a pending state.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadOnlyMany
  # ReadWriteOnce
  # ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```

When a PVC is deleted, you can choose what happens to the PV, such as retain the volume by default until it is manually deleted later.

```yaml
persistentVolumeReclaimPolicy: Retain / Delete / Recycle
```

## Storage Class

Storage Classes can be used in a PVC definition file to automatically provision a PV using a provisioner such as AWS EBS, GCE Persistent Disk, Azure Disk etc.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: google-storage
  resources:
    requests:
      storage: 500Mi
```
