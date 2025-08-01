

# Kubernetes Cluster Setup For Ubuntu 22.04.5 LTS (Jammy Jellyfish)




<br>


---

## Kubernetes Quickstart Cluster Implementation for Ubuntu 

### Introduction
Kubernetes is a powerful open-source system for automating the deployment, scaling, and management of containerized applications. It provides a platform for managing distributed systems, ensuring high availability, scalability, and reliability. This repository provides a step-by-step guide for setting up a **Kubernetes cluster on Rocky Linux**, making it easier for developers and system administrators to deploy and manage their Kubernetes environment.

### Purpose
This guide is intended for users who want to set up a Kubernetes cluster on **Rocky Linux**. It covers the installation and configuration of essential Kubernetes components, such as **kubelet**, **kubeadm**, and **kubectl**, as well as the **containerd runtime** and the necessary network add-ons. By following this guide, you will be able to quickly deploy a fully functional Kubernetes cluster suitable for both development and production environments.

### Prerequisites
Before you begin, ensure that you have:
- A **Rocky Linux** system (Master and Worker nodes)
- A minimum of **2GB RAM** and **2 CPUs** for each node
- A clean installation of **Rocky Linux 8** (or similar version)
- **Root or sudo privileges** on each node
- Basic knowledge of Linux commands and Kubernetes concepts

### Features
- **Quick Setup**: This guide provides a simplified process to quickly get Kubernetes up and running on Rocky Linux.
- **Master and Worker Node Setup**: Detailed steps for setting up the Kubernetes master and worker nodes.
- **Firewall Configuration**: Ensures necessary ports are open for seamless communication across nodes.
- **Flannel Networking**: Deploy the Flannel network add-on to enable communication between pods.
- **Kubernetes Dashboard**: Instructions on installing and accessing the Kubernetes Dashboard for easier management of the cluster.
- **Cluster Management**: Set up the tools and configuration for adding new nodes and managing the cluster.

### Overview of Kubernetes Components
- **kubeadm**: A tool for managing Kubernetes cluster lifecycle. It is used for bootstrapping the cluster by initializing the master node and joining worker nodes.
- **kubelet**: An agent running on each node in the cluster that ensures containers are running in pods.
- **kubectl**: A command-line tool for interacting with the Kubernetes API server and managing resources within the cluster.
- **containerd**: A container runtime used for running containers within the Kubernetes environment.
- **Flannel**: A simple and easy-to-use network overlay for Kubernetes, providing network connectivity to pods.

### Steps Included in This Guide
1. **Kernel Module Configuration**: Load the necessary kernel modules for container networking.
2. **Container Runtime Installation**: Install **containerd** as the container runtime.
3. **Kubernetes Installation**: Install Kubernetes components (`kubelet`, `kubeadm`, and `kubectl`) from the official Kubernetes repository.
4. **Cluster Initialization**: Initialize the Kubernetes master node and configure the network.
5. **Worker Node Joining**: Join the worker nodes to the cluster using the join command.
6. **Network Add-On**: Install Flannel for pod networking across the nodes.
7. **Kubernetes Dashboard**: Set up a web-based UI for monitoring and managing the cluster.

### Conclusion
By the end of this Implementation, you will have a fully functional Kubernetes cluster running on **Rocky Linux**. This setup will serve as the foundation for deploying containerized applications, scaling them, and managing them efficiently.


---









<br>


<br>




# ------------------  Implementation Steps ------------------


<br>


<br>







Below is the organized step-by-step Implementation guide for setting up Kubernetes Cluster.

---

# **Common Installation Steps (For Both Master and Worker Nodes)**

1. **Enable Kernel Modules for Kubernetes**
   ```yml
   sudo tee /etc/modules-load.d/containerd.conf <<EOF
   overlay
   br_netfilter
   EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```

2. **Configure Kernel Parameters**
   ```yml
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.ipv4.ip_forward = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   EOF

   sudo sysctl --system
   ```

3. **Disable Swap**
   ```yml
   sudo swapoff -a
   sudo sed -i '/swap/d' /etc/fstab
   ```

4. **Install and Configure Containerd Runtime **
   ```yml
   apt install -y containerd
   mkdir -p /etc/containerd
   containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
   sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

   systemctl restart containerd
   systemctl enable containerd

   
   ```

5. **Add Kubernetes Components**
   ```yml
   apt install -y curl gnupg2 apt-transport-https ca-certificates software-properties-common

   mkdir -p /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

   apt update
   apt install -y kubelet kubeadm kubectl
   apt-mark hold kubelet kubeadm kubectl

   ```

---


<br>

# **Master Node Setup**



1. **UFW Port Allowing**
     - If UFW is enabled, allow necessary ports:
   ```yaml
   ufw allow 6443/tcp        # API server
   ufw allow 2379:2380/tcp   # etcd
   ufw allow 10250/tcp       # Kubelet API
   ufw allow 10251/tcp       # kube-scheduler
   ufw allow 10252/tcp       # kube-controller-manager
   ufw allow 10257/tcp       # etcd metrics
   ufw allow 10259/tcp
   ufw allow 179/tcp         # Calico/Flannel BGP
   ufw allow 4789/udp        # VXLAN for Flannel
   ufw reload

   ```


2. **Initialize Kubernetes Cluster**

```yml
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<Master_Node_IP>
```

3. **Save the Join Command**
   - Copy the token displayed at the end of the `kubeadm init` command.

4. **Set Up kubeconfig**
   ```yaml
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config

   #OR (for root)

   echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
   source ~/.bashrc

   ```

5. **Install Pod Network Add-on (Flannel)**
   ```yaml
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

   ```

6. **Retrieve Join Token (if not saved earlier)**
   ```yaml
   kubeadm token create --print-join-command
   ```

7. **Verify Cluster Status**

```yml
   kubectl get nodes
   kubectl get pods -A
```

---

<br>

# **Worker Node Setup**

1. **Configure Firewall Rules**
   ```yaml
    # Allow TCP ports 179, 10250, and 30000-32767
    ufw allow 179/tcp
    ufw allow 10250/tcp
    ufw allow 30000:32767/tcp

   # Allow UDP port 4789
    ufw allow 4789/udp

   # Reload UFW to apply changes
    ufw reload

   ```

2. **Join the Cluster**
   - Run the join command copied from the master node after initialization.

---



## Example:


When you run the command `kubeadm token create --print-join-command` on the master node, the output will look similar to the following:

```yml
kubeadm join <Master_Node_IP>:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
```

### **Explanation of the Output**:
1. **kubeadm join**: The command to join a worker node to the master node.
2. **192.168.1.10:6443**: The IP address of the master node (`192.168.1.10` in this example) and the Kubernetes API server port (`6443`).
3. **--token abcdef.0123456789abcdef**: The token used to authenticate the worker node with the master node.
4. **--discovery-token-ca-cert-hash sha256:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890**: The hash of the CA certificate used to verify the master node's identity.

This output is the exact command you will run on the worker node to join it to the cluster.




---



<br>
<br>




### **Install Kubernetes Dashboard (Master Node Only  & its Optional )**

1. **Deploy the Dashboard**
   ```yaml
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
   ```

2. **Access the Dashboard**
   ```yaml
   kubectl proxy
   ```
   - Access via: `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

3. **Create Dashboard Service Account**
   - Create a YAML file (`service-account.yaml`) with the following content:
     ```yaml
     apiVersion: v1
     kind: ServiceAccount
     metadata:
       name: dash-admin
       namespace: kube-system
     ```
   - Apply it:
     ```yaml
     kubectl apply -f service-account.yaml
     ```

4. **Create ClusterRoleBinding**
   - Create a YAML file (`cluster-role-binding.yaml`) with the following content:
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: ClusterRoleBinding
     metadata:
       name: dash-admin
     roleRef:
       apiGroup: rbac.authorization.k8s.io
       kind: ClusterRole
       name: cluster-admin
     subjects:
     - kind: ServiceAccount
       name: dash-admin
       namespace: kube-system
     ```
   - Apply it:
     ```yaml
     kubectl apply -f cluster-role-binding.yaml
     ```

5. **Generate Access Token**
   ```yaml
   kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep dash-admin | awk '{print $1}')
   ```
   - Copy the token and paste it into the dashboard login page.

---

This structured implementation separates the steps clearly for the master and worker nodes, ensuring clarity and ease of execution.













<br>
<br>
<br>
<br>



**👨‍💻 𝓒𝓻𝓪𝓯𝓽𝓮𝓭 𝓫𝔂**: [Suraj Kumar Choudhary](https://github.com/Surajkumar4-source) | 📩 **𝓕𝓮𝓮𝓵 𝓯𝓻𝓮𝓮 𝓽𝓸 𝓓𝓜 𝓯𝓸𝓻 𝓪𝓷𝔂 𝓱𝓮𝓵𝓹**: [csuraj982@gmail.com](mailto:csuraj982@gmail.com)





<br>

























