
# Setting Up a Kubernetes Cluster with CRI-O

This document outlines the steps to set up a Kubernetes cluster using Kubernetes v1.31 and CRI-O v1.31 as the container runtime.



## Prerequisites

- A Linux-based operating system (e.g., Ubuntu, Debian). 
- Root or sudo access to the machine. 
- A minimum of 2 CPUs and 2GB of RAM is recommended. 
- Disable swap memory.
## Steps

#### 1. Set Environment Variables
Set the environment variables for Kubernetes and CRI-O versions.

```bash
export KUBERNETES_VERSION=v1.31
export CRIO_VERSION=v1.31
```

#### 2. Add Kubernetes APT Repository
- Download the Kubernetes signing key and save it:

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- Add the Kubernetes APT repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

#### 3. Add CRI-O APT Repository
- Download the CRI-O signing key and save it:
```bash
sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
```

- Add the CRI-O APT repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | \
sudo tee /etc/apt/sources.list.d/cri-o.list
```

#### 4. Install Packages
Update the package list and install CRI-O, kubelet, kubeadm, and kubectl.

```bash
sudo apt update
sudo apt-get install -y cri-o kubelet kubeadm kubectl
```


#### 5. Configure Networking
- Disable swap (important for Kubernetes):
```bash
sudo swapoff -a
```

- Load the necessary kernel modules:
```bash
sudo modprobe br_netfilter
```

- Enable IP forwarding:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

- Configure sysctl parameters for Kubernetes networking:
To persist these settings across reboots, add the following lines to /etc/sysctl.conf:

```bash
echo "net.bridge.bridge-nf-call-ip6tables=1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
```

- Apply the changes
```bash
sudo sysctl -p
```

#### 6. Setup Hostnames for Nodes
Edit the /etc/hosts file on each node to map the hostnames to their respective IP addresses. For example:

```bash
sudo vim /etc/hosts
```

Add lines similar to the following (replace <master-ip> and <worker-ip> with the actual IP addresses of your nodes):

```bash
<master-ip> master-node
<worker-ip1> worker-node-1
<worker-ip2> worker-node-2
```

#### 7. Configure CRI-O
Edit the CRI-O configuration file to set the cgroup manager.

```bash
sudo vim /etc/crio/crio.conf
```

Add or modify the following lines in the [crio.runtime] section:
```bash
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"
```


#### 8. Initialize the Kubernetes Cluster
Run the kubeadm command to initialize the cluster. Make sure to specify the pod network CIDR (here we use Flannel’s default):

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```


Now Set Up kubectl for the Non-Root User
If you want to use kubectl as a non-root user, run the following commands:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


#### 9. Deploy a Pod Network Add-On
To allow pods to communicate with each other, you need to deploy a pod network add-on. Here, we will use Flannel:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```



#### 10. Join Worker Nodes to the Cluster
After initializing the Kubernetes master node, you need to add the worker nodes to the cluster. To do this, you will generate a join command that contains a token and the IP address of the master node. This command will be run on each worker node.


```bash
kubeadm token create --print-join-command
```

This command generates a join command similar to the following:

```bash
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```


#### Note: It is essential to run Steps 1 through 7 on each worker node before proceeding to join the cluster. This step is mandatory to ensure proper configuration and functionality of the cluster.

Copy the generated command.

SSH on worker Nodes

Run the join command on each worker node:

