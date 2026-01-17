Here is the consolidated guide formatted into a clean, professional, and scannable Markdown file. I have organized the technical commands into appropriate syntax-highlighted blocks and used tables to clarify the infrastructure layout.

---

# Full Kubernetes Cluster Setup Guide: GMKtec K8 Plus

**Target Stack:** Agentic AI, Kafka, Flink, Cassandra, MongoDB, Vector DB.

This guide provides the exact sequence for transitioning from a Proxmox base to a functioning Kubernetes cluster. We will utilize `local-lvm` for VM disks and `local` storage for ISO images.

---

## 1. Prepare the Ubuntu ISO

1. Log in to your **Proxmox Web GUI** (e.g., `https://your-ip:8006`).
2. Select your node (e.g., `pve`) on the left, then click on the **local** storage.
3. Go to **ISO Images** -> **Download from URL**.
4. Paste the Ubuntu 24.04 Server ISO link:
`https://releases.ubuntu.com/24.04/ubuntu-24.04.1-live-server-amd64.iso`
5. Click **Query URL** then **Download**.

---

## 2. Create the "Golden" Template VM

Instead of installing Ubuntu three times, we create one "Golden Image" to ensure identical configurations across all nodes.

### Step-by-Step VM Creation:

* **General:** Name it `ubuntu-template`.
* **OS:** Select the Ubuntu ISO from `local` storage.
* **System:** Keep defaults (ensure **Qemu Agent** is checked).
* **Disks:** * Storage: `local-lvm`.
* Size: `100 GB`.


* **CPU:** `4 Cores`, Type: `host` (Crucial for AI/Data workload performance).
* **Memory:** `4096 MB`.
* **Network:** Default `vmbr0`.

### Finalize the Base OS:

Start the VM and complete the installer. **Do not select any Featured Server Snaps.** Once rebooted, log in and run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y qemu-guest-agent curl apt-transport-https ca-certificates

```

**Stop:** Shut down the VM. Right-click it in Proxmox and select **Convert to Template**.

---

## 3. Deploy the K8s Nodes

Right-click your `ubuntu-template` and select **Clone** for each node. Use **Full Clone**.

| Name | ID | RAM (Rec.) | vCPUs | Purpose |
| --- | --- | --- | --- | --- |
| **k8s-master** | 101 | 8 GB | 4 | Control Plane & Management |
| **k8s-worker-data** | 102 | 40 GB | 6 | Kafka, Cassandra, MongoDB, Flink |
| **k8s-worker-ai** | 103 | 32 GB | 6 | Vector DB, Agentic AI, Microservices |

### Post-Clone Disk Resizing:

1. Select **k8s-worker-data** -> Hardware -> Hard Disk -> Disk Actions -> **Resize**. Add `700 GB`.
2. Select **k8s-worker-ai** -> Hardware -> Hard Disk -> Disk Actions -> **Resize**. Add `500 GB`.

---

## 4. Ubuntu Post-Install Configuration

Run these commands on **all three VMs** to prepare for Kubernetes networking requirements.

```bash
# 1. Disable Swap (Required for Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Load Kernel Modules for K8s Networking
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 3. Set Sysctl for Networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

```

---

## 5. Install Container Runtime (Containerd)

Kubernetes requires a runtime to manage containers. Run this on **all nodes**:

```bash
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup for better resource management
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd

```

---

## 6. Install Kubernetes Tools (Kubeadm, Kubelet, Kubectl)

Run this on **all nodes** to install the core orchestration tools:

```bash
# Download the Google Cloud public signing key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install tools
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

---

**Your VMs are now ready for initialization.**

**Now we run the specific `kubeadm init` commands and perform Cilium CNI setup to link these three VMs into a single cluster?**



This stage will turn your our separate VMs into a unified "brain" and "body" system. We will perform the initialization on the **Master**, join the **Workers**, and then lay down the **Cilium** networking layer.

Copy and paste the following into your documentation or terminal as you go.

---

## 1. Initialize the Master Node (VM0)

Run this only on the **k8s-master** node. We are using the `10.244.0.0/16` CIDR which is a standard compatibility range for most networking plugins.

```bash
# Initialize the cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --node-name k8s-master

# Configure your local user to manage the cluster
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

**IMPORTANT:** At the end of the output, you will see a line starting with `kubeadm join ...`. **Copy this entire line.** You will need it for the workers in the next step.

---

## 2. Install Cilium CNI (Networking)

Kubernetes nodes cannot communicate until a Pod Network (CNI) is installed. We are using **Cilium** because it uses eBPF technology, which is significantly faster for high-throughput tools like Kafka and Flink.

Run this on the **Master**:

```bash
# Download Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz"
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz

# Install Cilium into the cluster
cilium install

```

---

## 3. Join Worker Nodes

Now, go to **k8s-worker-data** and **k8s-worker-ai**. Paste the join command you copied from Step 1. It should look something like this:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>

```

---

## 4. Verification

Back on the **Master**, check if the nodes are visible and "Ready" (it may take 60 seconds for Cilium to initialize each node):

```bash
kubectl get nodes

```

**Expected Output:**

```text
NAME              STATUS   ROLES           AGE   VERSION
k8s-master        Ready    control-plane   5m    v1.30.x
k8s-worker-data   Ready    <none>          2m    v1.30.x
k8s-worker-ai     Ready    <none>          2m    v1.30.x

```

---

## 5. Labeling Your Workers (Strategic Step)

Since you have specific hardware intentions for these nodes, let’s label them. This ensures that when you deploy Kafka or your Vector DB later, Kubernetes knows exactly where to put them.

```bash
# Mark the data worker for DBs and Streaming
kubectl label node k8s-worker-data capability=data-intensive

# Mark the AI worker for LLMs and Agents
kubectl label node k8s-worker-ai capability=ai-compute

```

---

**Next Step:** Your cluster is now "Alive." Would you like me to show you how to install **Longhorn** storage? Since you have a 2 TB NVMe, Longhorn is essential because it allows your databases to persist data safely even if a pod restarts.
