############################		How to Install Kubernetes Cluster on Ubuntu 22.04	##############################

# Prerequisites

# Before diving into the installation, ensure that your environment meets the following prerequisites:

#    An Ubuntu 22.04 system.
#    Privileged access to the system (root or sudo user).
#    Active internet connection.
#    Minimum 2GB RAM or more.
#    Minimum 2 CPU cores (or 2 vCPUs).
#    20 GB of free disk space on /var (or more).

# Step 1: Update and Upgrade Ubuntu (all nodes)

# Begin by ensuring that your system is up to date. Open a terminal and execute the following commands:

sudo apt update
sudo apt upgrade

# Step 2: Disable Swap (all nodes)

# To enhance Kubernetes performance, disable swap and set essential kernel parameters. Run the following commands on all nodes to disable all swaps:

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Step 3: Add Kernel Parameters (all nodes)

# Load the required kernel modules on all nodes:

sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure the critical kernel parameters for Kubernetes using the following:

sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Then, reload the changes:

sudo sysctl --system

# Step 4: Install Containerd Runtime (all nodes)

# We are using the containerd runtime. Install containerd and its dependencies with the following commands:

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Enable the Docker repository:

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Update the package list and install containerd:

sudo apt update
sudo apt install -y containerd.io


# Configure containerd to start using systemd as cgroup:

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Restart and enable the containerd service:

sudo systemctl restart containerd
sudo systemctl enable containerd


# Step 5: Add Apt Repository for Kubernetes (all nodes)

# Kubernetes packages are not available in the default Ubuntu 22.04 repositories. Add the Kubernetes repositories with the following commands:

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list


# Step 6: Install Kubectl, Kubeadm, and Kubelet (all nodes)

#After adding the repositories, install essential Kubernetes components, including kubectl, kubelet, and kubeadm, on all nodes with the following commands:

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl



# Step 7: Initialize Kubernetes Cluster with Kubeadm (master node)

# With all the prerequisites in place, initialize the Kubernetes cluster on the master node using the following Kubeadm command:

kubeadm init --control-plane-endpoint 192.168.1.126:6443        # Your Load Balancer IP

# After the initialization is complete make a note of the kubeadm join command for future reference.

# Run the following commands on the master node as a regular user :

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Next, use kubectl commands to check the cluster and node status:


kubectl get nodes


# Step 8: Add Master and  Worker Nodes to the Cluster

# On each worker and master node, use the kubeadm join command you noted down earlier:

# CONTROL PLANE:
# You can now join any number of control-plane nodes by copying certificate authorities
# and service account keys on each node and then running the following as root:

  kubeadm join 192.168.1.126:6443 --token 43mw6f.f0260b1qswxntz1r \
	--discovery-token-ca-cert-hash sha256:052375ac6129c49e4cbce9251d9cc72c3846b1c89dafd448c3ee77ecdbbe0419 \
	--control-plane 

# WORKER NODE:
# Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.126:6443 --token 43mw6f.f0260b1qswxntz1r \
	--discovery-token-ca-cert-hash sha256:052375ac6129c49e4cbce9251d9cc72c3846b1c89dafd448c3ee77ecdbbe0419 



# Generate new token to join work node
# Use command below to generate the join command for work node.

kubeadm token create --print-join-command


# How to label a node as worker

kubectl label nodes <nombre-del-nodo> node-role.kubernetes.io/worker=worker



# Step :9 Install Kubernetes Network Plugin (master node)

# To enable communication between pods in the cluster, you need a network plugin. Install the Calico network plugin with the following command from the master node:

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml



# Step 10: Verify the cluster and test (master node)

# Finally, we want to verify whether our cluster is successfully created.

kubectl get pods -n kube-system
kubectl get nodes


