KubeADM is a command-line tool used to bootstrap, initialize, and manage Kubernetes clusters. It is part of the Kubernetes project and is particularly helpful for simplifying the setup of a Kubernetes cluster, making it easier for administrators and developers to get started with Kubernetes.
Before you begin 
A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat.
2 GB or more of RAM per machine (any less will leave little room for your apps).
2 CPUs or more.
Full network connectivity between all machines in the cluster (public or private network is fine). Unique hostname, MAC address, and product_uuid for every node. 

We’ll use a Vagrant file for setting up the cluster

Download KubeADM from Kubernetes folder in our Knowledge Base. (You’ll find a vagrant file and necessary scripts for setting up the Environment) 

Clone this repository "https://github.com/flexyourcuriosity/kubeADM.git"

Creating VMs using Vagrant .... Run through the vagrant file which creates our Kubernetes cluster.
Run vagrant up (It takes a few minutes for spinning up the VMs). After completing run vagrant status. Three virtual machies spinned up one being the control plane and other being the minions.
Setting Up Kernel Modules and Sysctl Parameters for Kubernetes Networking
Configure kernel modules and sysctl parameters required for Kubernetes networking setup. These configurations are essential for enabling proper network communication and forwarding within a Kubernetes cluster.
Run this in all the three nodes,
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
This part writes the module names 'overlay' and 'br_netfilter' into the file /etc/modules-load.d/k8s.conf. These kernel modules are required for Kubernetes networking. After writing to the file, it uses sudo tee to both display the text (cat <<EOF) and save it to the specified file.
Load these modules using the below commands.
sudo modprobe overlay
sudo modprobe br_netfilter
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
This part sets certain system parameters required by Kubernetes. It writes the specified configuration into the file /etc/sysctl.d/k8s.conf.
net.bridge.bridge-nf-call-iptables and net.bridge.bridge-nf-call-ip6tables are set to 1 which enables iptables to see bridged traffic.
net.ipv4.ip_forward is set to 1 which enables IP forwarding, allowing packets to pass through the node.
# Apply sysctl params without reboot
sudo sysctl --system
Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay
These commands will list the loaded kernel modules and filter the output to show only the lines containing br_netfilter and overlay, respectively.

Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
Each line corresponds to the queried system variable followed by its current value. If the values are set to 1 as expected, then your system is properly configured. If any of the values are not 1, it means they are not configured as expected.
CONTAINER RUNTIME INSTALLATION (All the nodes)
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
# Add the repository to Apt sources:
echo \ "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \ "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
INSTALL
sudo apt-get install containerd.io 
Following these instructions will ensure a successful installation of Docker, including the installation of necessary dependencies and the Docker runtime Containerd.
Configuring systemD Cgroup Driver
sudo vi  /etc/containerd/config.toml  

Remove everything and add these lines,
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
Configuring Containerd to use the systemd cgroup driver (SystemdCgroup = true) ensures that Containerd integrates with the systemd service manager for managing container resources.
sudo systemctl restart containerd
Installing KubeADM, Kubelet, Kubectl (All the nodes)
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
Creating a cluster with kubeadm
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<Masternode IP>
 NOTE: The above command returns a command which is to be executed in the worker node, Here’s the output.
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
POD network configuration
After initialization, configure the pod network addon to enable networking between pods across nodes. In this demo we’ll use weave net  (https://www.weave.works/docs/net/latest/kubernetes/kube-addon/)
EXECUTE THIS IN MASTER !!
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
Editing Weave Net DaemonSet for IP Allocation Range
Now we need to edit the DS and add our CIDR Ip (10.244.0.0/16)
Identify Weave Net DaemonSet:
kubectl get ds -n kube-system
Edit the DaemonSet:
kubectl edit ds weave-net -n kube-system
Add this in ENV,
- name: IPALLOC_RANGE 
   value: 10.244.0.0/16                       (MAINTAIN INDENTATION )
Verification:
kubectl describe ds weave-net -n kube-system
Ensure that the specified IP allocation range (10.244.0.0/16) does not conflict with existing network configurations in your environment.
After configuring the POD Network you can execute the KUBEADM JOIN command which is returned when initializing kubeadm in the worker nodes
Using “kubeadm join” command on the minions we can connect to the control plane:
SSH into each worker node and execute the kubeadm join command provided by the kubeadm init output. 
Verification:
Verify that the node has successfully joined the cluster by running the following command on the control plane node:
kubectl get nodes
We’ve successfully bootstrapped our Kubernetes cluster using kubeADM. 
