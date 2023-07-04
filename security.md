# Security

## TLS Basics

Transport Layer Security (TLS) is a cryptographic protocol designed to provide communications security over a computer network.

Data sent over the internet is encrypted and sent with a key. However, this is Symmetric Encryption and the key can also be sniffed by a hacker; therefore, it is not as secure as Asymmetric Encryption.

Asymmetric encryption uses a private key (key) and a public key (lock). You encrypt the data using the public key and can only be decrypted using the private key which must not be shared with others. Therefore, a hacker only gets the public key and encrypted data and cannot decrypt it. To create a public and private key pair you can use `ssh-keygen`.

Over HTTPS, a hacker can even use their own server to mimic the website you are trying to log in and access. To validate the website, you can view the certificate and see who signed and issued it, usually a trusted certificate authority (CA). This validation should be done automatically by the browser and if it is a fake certificate, it will warn the user.

This is to validate public website certificates, but you can host your own CA or employ a private offering from one of the trusted CAs to secure internal websites and applications.

### Naming Conventions

- Public keys (certificate) - public.crt or public.pem
- Private keys - private.key or private-key.pem

## TLS in Kubernetes

All components in Kubernetes use a public and private key to communicate with each other, such as clients: the scheduler, controller manager and kube proxy to connect to the servers: kube API server and ETCD server.

To create certificates for a Kubernetes cluster, use tools such as EasyRSA or OpenSSL.

1. Firstly, create the CA certificates by generating the keys, a certificate signing request and then sign the certificate:

    ```bash
    openssl genrsa -out ca.key 2048

    openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

    openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
    ```

2. Next, generate the client certificates, e.g., for an admin user in the system masters group, using a similar process and signing with the CA key-pair:

    ```bash
    openssl genrsa -out admin.key 2048

    openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr

    openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
    ```

3. This process can be repeated for the other Kubernetes components including the API server, but for all of the alias of the API server, they can be specified in the `[alt_names]` section in a `openssl.cnf` config file, passing this file into the command:

    ```bash
    openssl genrsa -out apiserver.key 2048

    openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf

    openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt
    ```

## View Certificate Details

As shown previously, the certificates can be generated manually, or an automated provisioning tool such as kubeadm can be used.

To decode a certification and view details:

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

It is important to check the names and alternate names, organisation, issuer and expiration date.

It can be helpful to also look at the service or kubeadm logs:

```bash
# Inspect service logs
journalctl -u etcd.service -l

# Inspect kubeadm logs
kubectl logs etcd-master

# If API or ETCD are down kubectl cannot be used so view logs in underlying docker container
docker ps -a | grep <kube-apiserver or etcd>
docker logs <container-id>
```

## Certificates API

Whenever a new user wants to have access to the cluster they need a certificate signing request (CSR) so the CA can validate their credentials with the appropriate privileges.

The CA files need to be protected and stored in a safe environment such as a secure server. Therefore, to create a new user, you need to log into this server each time.

This method of signing CSRs is manual. However, with the Certificates API, you can sign the request from the server using an API call by creating a CSR object, reviewing and approving that request then sharing the certificates to the new user.

This CSR object is created using a definition YAML file like any other object, as follows:

```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: user
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request:
    <Add the request here using command 'cat user.csr | base64 | tr -d "\n"'>
```

To create, view and then approve the CSRs use the following commands:

```bash
kubectl create -f <csr.yaml>

kubectl get csr

kubectl certificate approve <user>
```

To reject and delete unwanted CSRs use these commands:

```bash
kubectl certificate deny <user>

kubectl delete csr <user>
```

## KubeConfig

By default, Kubernetes looks for a KubeConfig file in the $HOME/.kube/config directory. Therefore, we don't need to specify any options for `kubectl` commands so far.

You can specify clusters, contexts (links clusters to users) and users in the KubeConfig file, using `current-context` to specify a default context.

```yaml
apiVersion: v1
kind: Config
clusters:
- name: example-cluster
  cluster:
    certificate-authority: ca.crt
    server: https://example-cluster:6443
contexts:
- name: example-user@example-cluster
  context:
    cluster: example-cluster
    user: example-user
    namespace: default
users:
- name: example-user
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

To view the current config file and change the context, use the following command:

```bash
kubectl config view

kubectl config use-context <example-user@example-cluster> --kubeconfig <path-to-config-file>
```

## API Groups

All resources are grouped into different API groups such as `/apps`, `/certificates.k8s.io` etc. under the named `/apis` group under the core API group. Under each group the resources have a set of actions called verbs.

Use the following command to list all resources and their API group and version:

```bash
kubectl api-resources
```

## Authorisation

We can authorise users to only access certain resources using different types of authorisation methods:

- Node
- Attribute Based Access Control
- Role Based Access Control
- Webhook

## Role Based Access Controls (RBAC)

We can restrict users based on their role (such as developers) by defining a role. We can create a `Role` as an object using a definition YAML file as follows:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

The next step is to link the user to the role using a `RoleBinding` object as shown:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

To list roles and role bindings and describe them, run the following commands:

```bash
kubectl get roles

kubectl get rolebindings

kubectl describe role developer

kubectl describe rolebinding devuser-developer-binding
```

To check if you have access to perform a particular action on a resource, use the command:

```bash
kubectl auth can-i <action> <object-type>/<object-name> --as <user> --namespace <namespace>

# Example
kubectl auth can-i delete pod/webapp --as dev-user --namespace test
```

## Cluster Roles and Role Bindings

Resources are categorised as either namespaced or cluster-scoped. For example, roles and role bindings are created in the namespace you specified or the default namespace if not specified.

To authorise users for cluster-wide resources such as nodes and persistent volumes (PV), we can use cluster roles and cluster role bindings. These are created using a definition file in a similar manner as previous for namespaced roles and role bindings.

You can create a cluster role for namespaces as well, the user will then have access to that resource across all namespaces.

```bash
kubectl get clusterroles | wc -l # count no. of roles

kubectl get clusterrolebindings | wc -l # count no. of role bindings

kubectl describe clusterrole <role-name>

kubectl describe clusterrolebinding <role-binding-name>
```

## Service Accounts

User accounts are used by humans such as an admin or developer and Service accounts are used by machines such as tools like Prometheus or Jenkins.

Service accounts are used to authenticate tools to access the Kubernetes cluster via the API server.

To create, list and describe a service account:

```bash
kubectl create serviceaccount <sa-name>

kubectl get serviceaccount

kubectl describe serviceaccount <sa-name>
```

When creating the service account, it automatically creates a token as a secret object, which stores the token. To create and view the token, use the command:

```bash
kubectl describe secret <sa-token-name>
```

To manually create a token from a service account:

```bash
kubectl create token <sa-name>
```

When making a call to the Kubernetes API, this token can be provided to the third-party application or service to authenticate it.

If the application or service is running on the cluster, this process can be made simple by mounting the token as a volume on the pod hosting the application as follows:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp
  serviceAccountName: myapp-sa
```

## Image Security

Security needs to be applied when pulling images from a repository like Docker Hub.

You can choose to make a repository private so you need credentials to access it.

```bash
docker login private-repository.io

docker run private-repository.io/apps/internal-app
```

For a pod to access the image from a private repository, use the full path to the image and the `imagePullSecrets` property:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: private-repository.io/apps/internal-app
  imagePullSecrets:
  - name: reg-cred
```

To create the secret, use the following command:

```bash
kubectl create secret docker-registry reg-cred \
  --docker-email=tiger@acme.example \
  --docker-username=tiger \
  --docker-password=pass1234 \
  --docker-server=private-repository.io
```

## Security Contexts

You can configure Security context under the `spec` section for the pod or for containers under the `container` section on the Pod definition file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  # Pod Level
  securityContext:
    runAsUser: 1000
  containers:
  - name: nginx
    image: nginx
    # Container Level
    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"]
```

Capabilities are only supported at the Container level and not at the Pod level.

To check the user and id inside a pod use the following commands:

```bash
kubectl exec <pod-name> -- whoami

kubectl exec <pod-name> -- id
```

## Network Policy

We can protect our resources by specifying inbound (ingress) and outbound (egress) rules for traffic using a network policy.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    # Select individual pods
    - podSelector:
        matchLabels:
          name: api-pod
    # Select all pods in a namespace
      namespaceSelector:
        matchLabels:
          name: prod
    # Select IP address ranges
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

## Kubectx

This tool is used to quickly switch contexts between clusters in a multi-cluster environment.

To install kubectx:

```bash
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
```

To list all contexts:

```bash
kubectx
```

To switch to a new context:

```bash
kubectx <context-name>
```

To switch back to the previous context:

```bash
kubectx -
```

To see current context:

```bash
kubectx -c
```

## Kubens

This tool is used to quickly switch between namespaces.

To install kubens:

```bash
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
```

To switch to a new namespace:

```bash
kubens <namespace-name>
```

To switch back to the previous namespace:

```bash
kubens -
```
