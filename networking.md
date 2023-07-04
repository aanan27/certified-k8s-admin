# Networking

## Environment

List the network interfaces on the node:

```bash
ip address
```

To show the bridge interface created by the container runtimes:

```bash
ip address show type bridge
```

To find out the default gateway IP address of a node:

```bash
ip route
```

Check the ports of a service

```bash
netstat -npl | grep -i <service-name>
```

## Pod Networking

Create a virtual network cable (veth pair)

```bash
ip link add <...>
```

Attach veth pair:

```bash
ip link set <...>
ip link set <...>
```

Assign IP address:

```bash
ip -n <namespace> addr add <...>
ip -n <namespace> route add <...>
```

Bring up interface:

```bash
ip -n <namespace> link set <...>
```

Delete veth pair:

```bash
ip link delete <...>
```

## Container Networking Interface (CNI)

CNI is used to automatically add containers to the network. The CNI plugin is configured in the Kubelet service on each node in the cluster. Kubelet looks at the `/etc/cni/net.d` configuration files.

Instead of our own script to connect nodes, we can use a CNI plugin such as Weaveworks.

Weave will deploy agents to all nodes that will work to target packets to be delivered to other nodes.

To install CNI plugin Weaveworks Weave Net as a pod in the cluster, use the following command to deploy weave networking solution:

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

## IP Address Management (IPAM) using Weave

The CNI plugin is responsible for assigning IP addresses to the containers. The IP addresses can be stored in a list using built-in plugins such as DHCP and host-local under the `ipam` section in `/etc/cni/net.d/net-script.conf`.

Different solutions may do it differently, for Weave, by default it assigns 10.32.0.0/12 to the entire network by default which gives around 1 million IP addresses for the network (between 10.32.0.1 to 10.47.255.254). From this range, the peers decide to split these addresses equally and assign one portion to each node and pods created in each node will have addresses in each respective portion.

## Service Networking

You rarely connect pods directly, you use services for pods to access each other, such as using `NodePort`. Even external users and applications have access to this service. For cluster-wide pod you can use `ClusterIP`.

Each `kubelet` service on each node watches for changes in the cluster via the `kube-apiserver` and every time a new pod needs to be created, it creates the pod on the nodes. It invokes the CNI plugin to configure networking on that pod.

Each node also runs `kube-proxy` which watches changes in the cluster and each time a new service is to be created, it deploys this but a service is cluster wide and not tied to the node. There is no namespace for services, it is a virtual object, it can be assigned an IP address from the `kube-proxy` where it can create forwarding rules. Therefore, if a pod reaches this address it gets forwarded to the pod.

Look for the IP address range for services `kube-api-server --service-cluster-ip-range ipNet`, by default it is 10.0.0.0/24.

```bash
ps aux | grep kube-api-server
```

You can see the forwarding rules created by the `kube-proxy` in the IP tables NAT output:

```bash
iptables -L -t nat | grep <service>
```

Also, to view the kube-proxy logs:

```bash
cat /var/log/kube-proxy.log
```

### Services vs Pods IP Ranges

To view IP Range for **services** within a cluster, look at the `kube-apiserver`:

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system
# OR
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

To view IP Range for **pods** within a cluster, look at the CNI plugin such as Weave:

```bash
kubectl describe pod weave-net-6d8jm -n kube-system
# OR
kubectl logs weave-net-6d8jm -n kube-system
```

## CoreDNS in Kubernetes

All pods and services can access each other using their IP addresses. Kubernetes creates a DNS record of each service using its name for e.g. "web-service", which is reachable in the same namespace. Otherwise, you will need to specify the namespace, type and cluster at the end such as:

```bash
curl http://web-service.namespace.svc.cluster.local
```

Records for pods are not created by default. We can enable this DNS record creation so records are created automatically where the IP address becomes the name using dashes (-) such as:

```bash
curl http://10-244-2-5.namespace.pod.cluster.local
```

The DNS service in Kubernetes called CoreDNS is a server used to watch the cluster for any new service or pods adn each time one is created, it adds a record for it in its database.

CoreDNS can be configured using the file: `/etc/coredns/Corefile`.

For pods/services to point to the CoreDNS server, it also uses a service named `kube-dns` by default.

The `/etc/resolv.conf` file can be used to create search entries for any name of the services, but not pods, you need the full name for the pod still.

## Ingress Networking

To make an application accessible to the public, you use a `NodePort` service with a specific port, so they can access it at the node IP address and port number (<http://node-ip:38080>).

In reality, we will configure the DNS server to point to this IP address so the users can access using the domain name instead (<http://application.com:38080>).

To remove service node ports, you use a proxy server (<http://application.com>). All of this works if the application is hosted on-premise. If your application is on a cloud service provider, you can change the NodePort service to `LoadBalancer` to send requests to the cloud provider to request a load balancer (paid service from the cloud).

If you have multiple services, you can use an **Ingress Controller**  (entering) such as deploying an Nginx Reverse Proxy, and configure rules on it called **Ingress Resources**. These are created using YAML definition files. Kubernetes clusters do not come with an Ingress Controller by default, so you must deploy one. There are a few solutions:

- GCE Load Balancer (supported)
- Nginx Reverse Proxy (supported)
- Contour
- HAProxy
- Traefik

### Ingress Controller

We use a deployment to deploy the Nginx Ingress Controller image, a service to expose it, a config map to feed Nginx configuration data and a service account for authorisation with the right permissions to access these objects.

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
      args:
      - /nginx-ingress-controller
      - --configmap=$(POD_NAMESPACE)/nginx-configuration
      env:
      - name: POD_NAME
        valueFrom:
          fieldRef:
            fieldPath: metadata.name
      - name: POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      ports:
      - name: http
        containerPort: 80
      - name: https
        containerPort: 443
```

The ConfigMap here helps to modify the Nginx configuration at a later date:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
```

Also, we need to create a service to link to the Ingress controller deployment:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  selector:
    name: nginx-ingress
```

Finally, we need to create a service account with the correct permissions (roles and role bindings).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

### Ingress Resource

Ingress resources are a set of rules and configurations applied on the Ingress controller, for example, you can configure rules to forward all incoming traffic to a single application.

An Ingress resource is created using a YAML definition file as shown, where there are multiple rules defined to route to two services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-example
spec:
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

You can also split traffic by host name:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-example
spec:
  rules:
  - host: app1.website.com
    http:
      paths:
      - backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.website.com
    http:
      paths:
      - backend:
          service:
            name: app2-service
            port:
              number: 80
```

We can also create Ingress resources imperatively using the command:

```bash
kubectl create ingress <ingress-name> --rule="host/path=service:port"
```

For different namespace apps, we need to ensure the correct path is directed, we can use a `rewrite` annotation to do this:

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```

This directs the traffic to the correct service "example-app.svc" when a user enters "domain.com/example-app", instead of "example-app.svc/example-app".
