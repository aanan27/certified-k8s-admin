# Application Lifecycle Management

## Rolling Updates and Rollbacks

When you create a deployment it creates a rollout and a new deployment revision e.g revision 1. When the application and container is updated, a new rollout is triggered and a new deployment revision is created e.g. revision 2.

To view the status of the rollout:

```bash
kubectl rollout status deployment/<deployment-name>
```

To see the revisions and history of the rollout:

```bash
kubectl rollout history deployment/<deployment-name>
```

### Deployment Strategy

- Recreate - Destroy all the instances of the application then deploy the new application instances, this causes some application downtime.

- Rolling Update - Default deployment strategy, it takes down and deploys a new instance one by one, to allow for seamless updates and minimal downtime.

You can use the `apply` and `set image` commands to update a deployment. When a deployment is updated, it creates a replica set automatically and the pods. When the application is updated, Kubernetes creates a new second replica set with the new pods and simultaneously takes down the old pods.

You can rollback a change using the command, which deletes the new pods and goes back to creating the old pods.

```bash
kubectl rollout undo deployment/<deployment-name>
```

## Commands and Arguments

A Dockerfile can be used to run commands and processes inside a container. These will be the container default commands and can be overwritten if commands are passed into the pod definition YAML file as shown below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example
    image: example-image
    command: ["command"]
    args: ["argument-1", "argument-2"]
    # OR
    command: ["command", "argument"]
    # OR
    command:
    - "command"
    - "argument"
```

## Environment Variables

You can pass in environment variables into the pod containers using the following methods outlined the in YAML definition file such as directly, using a ConfigMap or Secrets.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example
    image: example-image
    env:
    - name: ENV_VAR
      value: example
    # OR using ConfigMap
    env:
    - name: ENV_VAR
      valueFrom:
        configMapKeyRef:
    # OR using Secrets
    env:
    - name: ENV_VAR
      valueFrom:
        secretKeyRef:
```

You can create a ConfigMap using either an imperative or declarative approach using the following commands:

```bash
# Imperative
kubectl create configmap <config-name> --from-literal=<key>=<value>

kubectl create configmap <config-name> --from-file=<path-to-file>

# Declarative
kubectl create -f <config-map.yaml>
```

- config-map.yaml

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-config
    data:
      ENV_VAR: example
      ENV_VAR_2: example-2
    ```

To view configmaps use the following command:

```bash
kubectl get configmaps

kubectl describe configmaps
```

We can use this config-map.yaml file which declares the environment variables directly in our pod definition file using the `envFrom` property to inject as many environment variables as we want into our pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example
    image: example-image
    envFrom:
    - configMapRef:
        name: example-config
```

## Secrets

Secrets can be used to pass in secrets such as passwords or secret keys into a script. We can use the imperative or declarative approach to input secret data:

```bash
# Imperative
kubectl create secret generic <secret-name> --from-literal=<key>=<value>

kubectl create secret generic <secret-name> --from-file=<path-to-file>

# Declarative
kubectl create -f <secret-data.yaml>
```

- secret-data.yaml

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: example-secret
    data:
      SECRET_User: username
      SECRET_Password: password
    ```

For the declarative approach it is not safe to outline secrets in plain text format in the definition file, you have to describe them in an encoded format. To convert from plain text to an encoded format you can use the following Linux command:

```bash
echo -n 'username' | base64

echo -n 'password' | base64
```

To decode the encoded values use the decode option:

```bash
echo -n 'username (encoded)' | base64 --decode

echo -n 'password (encoded)' | base64 --decode
```

To view secrets and the associated encoded values:

```bash
kubectl get secrets

kubectl describe secrets

kubectl get secret example-secret -o yaml
```

We can inject secrets into the pods using the following `envFrom` property in the definition file to call on the secret-data.yaml file which declares the secrets:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example
    image: example-image
    envFrom:
    - secretRef:
        name: example-secret
```

Also, you can mount secrets in pods as volumes:

```yaml
volumes:
- name: secret-volume
  secret:
    secretName: example-secret
```

### Considerations

- Secrets are not encrypted, only encoded. Anyone can view the secrets file and decode it in a similar way to retrieve the secret. Therefore, do not upload secret objects to SCM along with code.
- Secrets are not encrypted in ETCD, so consider encrypting secret data at rest, see [below](#encrypting-secret-data-at-rest).
- Anyone able to create pods/deployments in the same namespace can access the secrets, so consider using least-privilege access to secrets such as Role Based Access Control (RBAC)
- Consider third-party secrets store providers such as AWS, Azure etc.

### Encrypting Secret Data at Rest

This section outlines how to enable and configure encryption of secrets at rest, for more information, click [here](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).

1. Install the ETCD client to run `etcdctl` commands:

    ```bash
    apt-get install etcd-client
    ```

2. Create a new encryption config file:

    ```yaml
    apiVersion: apiserver.config.k8s.io/v1
    kind: EncryptionConfiguration
    resources:
    - resources:
      - secrets
      - configmaps
      - pandas.awesome.bears.example
      providers:
      - aescbc:
          keys:
          - name: key1
            secret: <BASE 64 ENCODED SECRET>
      - identity: {}
    ```

3. Generate a 32-byte random key and encode it using base64 on Linux:

    ```bash
    head -c 32 /dev/urandom | base64
    ```

4. Place the key inside the `secret` field in the `EncryptionConfiguration` config file and set the `--encryption-provider-config` flag on the `kube-apiserver` to point to the location of that config file.

5. Restart the API server

6. Create a new secret called `secret1` in the `default` namespace:

    ```bash
    kubectl create secret generic secret1 -n default --from-literal=mykey=mydata
    ```

7. Use the `etcdctl` command to view the secret in an encrypted format (`hexdump -C`). The arguments are used to connect to the ETCD server:

    ```bash
    ETCDCTL_API=3 etcdctl \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key  \
   get /registry/secrets/default/secret1 | hexdump -C
    ```

## Multi Container Pods

You may need two microservices to work together, such as a web server and a log-in service. We can deploy a web server and log agent as separate entities, which can scale up and down together. They share the same pod, volume, network and application lifecycle. To create a multi container pod as below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-webapp
  labels:
    name: example-webapp
spec:
  containers:
  # Container 1
  - name: example-webapp
    image: example-webapp
  # Container 2
  - name: example-agent
    image: example-agent
```

To view a pod's logs:

```bash
kubectl logs <pod-name> -n <namespace>
```

## Init Containers

You may need a process to run until completion inside a container instead of running and staying alive at all times. The process shall only run one time when the pod is first created, this is known as initContainers, an example is shown below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone <some-repository-that-will-be-used-by-application> ; done;']
```
