# Cluster Design and Installation

To install a cluster you can use the following tools:

- Kubeadm for on-premise
- GKE for GCP
- kOps for AWS
- Azure Kubernetes Service (AKS) for Azure

## Infrastructure

Kubernetes can be applied on a local machine, physical or virtual servers on-premise or in the cloud.

You need to set up Kubernetes on Linux, if on Windows, you can set up a linux-based VM or Docker machine.

You can use **Minikube** to deploy a single node Kubernetes cluster using virtual machines created by VirtualBox. It deploys the VMs automatically.

The **Kubeadm** tool can deploy a single or multi-node cluster, you need to configure and provision the VMs, ensuring they are ready for the cluster to be deployed.

### Turnkey Solutions

Also, there are turnkey solutions where you can provision and configure VMs, use scripts to deploy the cluster and maintain the VM yourself as one solution. E.g. Kubernetes on AWS using kOps. A list of turnkey solutions to deploy and manage a private cluster in your organisation are as follows:

- OpenShift
- Cloud Foundry Container Runtime
- VMware Cloud PKS
- Vagrant

### Hosted Solutions

There are similar solutions but the VM is hosted, managed and maintained on a public cloud such as:

- Google Container Engine (GKE)
- OpenShift Online
- Azure Kubernetes Service
- Amazon Elastic Container Service for Kubernetes (EKS)

### High Availability

To ensure high availability it is best to have redundancy across every component in a cluster, even the master node and control plane components.

It is recommended to have a minimum of 3 instances in an ETCD cluster (3/2 + 1 = Quorum of 2) which gives a fault tolerance of 1. Odd numbers, 3, 5 or 7 are recommended.

## Installation

To install a Kubernetes cluster, you need to install Kubeadm on the master node and the worker nodes using the following steps as follows:

1. Install a container runtime such as Containerd, firstly configure the environment to forward IPv4 and let iptables see bridged traffic:

    ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter

    # sysctl params required by setup, params persist across reboots
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

    # Apply sysctl params without reboot
    sudo sysctl --system
    ```

2. To verify this was configured properly:

    ```bash
    lsmod | grep br_netfilter
    lsmod | grep overlay

    sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
    ```

3. Add the Docker repository and install Containerd using the following commands:

    ```bash
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg

    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt-get update

    sudo apt-get install containerd.io
    ```

4. Check the installation of Containerd was successful:

    ```bash
    systemctl status containerd
    ```

5. Now, we can configure the cgroup drivers, which are used to constain resources allocated to processes. There are two cgroup drivers available: cgroupfs and systemd. As long as the kubelet and container runtime use the same cgroup then they will work together. We can check our init system using process with ID 1:

    ```bash
    ps -p 1
    ```

6. Now if we edit the Containerd configuration file and replace its contents with the following to use the systemd cgroup driver:

    ```bash
    sudo vim /etc/containerd/config.toml
    ```

    ```toml
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
    ```

    ```bash
    sudo systemctl restart containerd
    ```

7. The commands below are used to install Kubeadm, Kubelet and Kubectl:

   - `kubeadm`: the command to bootstrap the cluster.

   - `kubelet`: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

   - `kubectl`: the command line util to talk to your cluster.

    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl

    curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

Now we have everything installed on the master and worker nodes, we can create and initialise our Kubernetes cluster by running the following command on the master node only.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<apiserver-eth0-private-ip>
```

Next, follow the steps and run the commands that appear in the output log from running the previous command.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

We can now deploy a pod network to the cluster using a networking addon such as Weavenet to connect our cluster components as follows:

```bash
# kubectl apply -f <pod-network-addon.yaml>
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

Finally, we can join the worker nodes to the cluster using the following command:

```bash
sudo kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
