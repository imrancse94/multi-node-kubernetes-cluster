# Kubernetes Cluster Setup on VirtualBox with 3 Nodes

This guide provides detailed instructions to set up a Kubernetes (k8s) cluster using VirtualBox. The cluster will consist of the following nodes:

1. **Master Node**
2. **Worker Node 1**
3. **Worker Node 2**

## Prerequisites
Ensure VirtualBox is installed and configured for the virtual machines. You can refer to the image below for the architecture:

![Kubernetes Cluster Architecture](https://github.com/imrancse94/multi-node-kubernetes-cluster/blob/main/images/image1.webp?raw=true)

---

## Step 1: Disable Swap on All Nodes
Disable swap to ensure Kubernetes functions correctly.

```bash
tswapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## Step 2: Enable IPv4 Packet Forwarding

### Configure sysctl Parameters

Create a configuration file to enable IPv4 forwarding:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```

### Apply Parameters Without Reboot

```bash
sudo sysctl --system
```

---

## Step 3: Verify IPv4 Packet Forwarding
Ensure IPv4 packet forwarding is enabled:

```bash
sysctl net.ipv4.ip_forward
```

---

## Step 4: Install Containerd

### Add Docker GPG Key and Repository

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository
echo \  \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \  \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \  \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install containerd.io -y
sudo systemctl enable --now containerd
```

---

## Step 5: Install CNI Plugin

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.0.tgz
```

---

## Step 6: Configure IPv4 and IPTables

### Load Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure sysctl for Network Settings

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
modprobe br_netfilter
sysctl -p /etc/sysctl.conf
```

---

## Step 7: Modify Containerd Configuration for systemd Support

Edit the `config.toml` file for containerd:

```bash
sudo nano /etc/containerd/config.toml
```

Paste the following configuration:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

Save and exit the editor.

---

## Step 8: Restart Containerd and Check Status

```bash
sudo systemctl restart containerd && sudo systemctl status containerd
```

---

## Step 9: Install Docker

```bash
sudo apt-get install docker.io -y
```

---

## Step 10: Access Kubernetes Repositories

### Add Kubernetes GPG Key

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### Add Kubernetes Repository

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes Components

```bash
sudo apt-get update
sudo apt install kubeadm kubelet kubectl -y
sudo apt-mark hold kubeadm kubelet kubectl
```

---

## Step 11: Initialize the Cluster

Pull required container images:

```bash
sudo kubeadm config images pull
```

Initialize the master node:

```bash
sudo kubeadm init --apiserver-advertise-address <master-ip> --control-plane-endpoint <master-ip> --ignore-preflight-errors Swap
```

---

## Step 12: Configure Networking with Calico

### Download and Edit Calico YAML

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/calico.yaml -O
```

Uncomment the line containing `CALICO_IPV4POOL_CIDR` to set the appropriate CIDR for your cluster.

### Apply the Configuration

```bash
kubectl apply -f calico.yaml
```

---

Your Kubernetes cluster is now set up and ready to use. Follow the instructions provided by `kubeadm init` to join the worker nodes to the cluster.
